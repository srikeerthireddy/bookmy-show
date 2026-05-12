# Async Order Processing Flow

## 1. Why async?

A synchronous payment call holds a database connection while it waits on an external gateway. That is the wrong place to spend connection time during a sale spike.

Using the same pool math:

connections held = (0.80 x RPS x 0.02) + (0.20 x RPS x 0.8)
connections held = 0.176 x RPS

At 500 max connections, the pool exhausts at about 2,841 RPS.

Now compare that with the noon burst. If 500,000 users arrive in 60 seconds, that is 8,333 RPS. Even if only 20 percent of those requests involve payment, that still means 1,667 payment RPS, which by itself would hold about 1,333 connections at 800 ms each. That is far beyond the pool ceiling.

So the API must stay fast: reserve seats, enqueue payment work, and return quickly.

## 2. Queue message format

Message body:

```json
{
  "bookingId": "2e0fbfb8-f0b3-4a17-9c7a-7d5bb1f4f9a1",
  "userId": "a5e3f2d1-2e1f-4f53-9c7b-8d9c4a4d0fbb",
  "eventId": 12345,
  "seatIds": [9001, 9002, 9003],
  "totalAmount": 7499.00,
  "currency": "INR",
  "paymentToken": "paytok_abc123",
  "idempotencyKey": "book-12345-user-abc-20260512-0001"
}
```

Field purpose:
- `bookingId`: the worker's primary DB target.
- `userId`: ties the payment and audit trail back to the caller.
- `eventId`: needed for seat and event validation.
- `seatIds`: the exact inventory the worker must confirm.
- `totalAmount`: prevents the worker from relying on a missing DB lookup for the amount.
- `currency`: required by the payment gateway and future-proofing for multi-currency support.
- `paymentToken`: the payment authorization handle from the API layer.
- `idempotencyKey`: guarantees the payment gateway and our own worker can retry safely without double-charging.

## 3. Worker logic

1. Read the SQS message and validate the JSON schema.
2. Start a database transaction and lock the booking row for the given `bookingId`.
3. If the booking is already terminal (`confirmed`, `failed`, or `refunded`), acknowledge the message and stop.
4. Verify the seat rows are still held by the same user and have not expired.
5. Call the payment gateway using the same `idempotencyKey`.
6. If the payment succeeds, update the booking to `confirmed`, mark each seat `booked`, clear hold metadata, commit, and acknowledge the message.
7. If the payment fails definitively, update the booking to `failed`, release each seat back to `available`, commit, and acknowledge the message.
8. If the payment times out or returns an ambiguous state, do not guess. Leave the booking pending, let SQS redeliver after visibility timeout, and retry with the same idempotency key.
9. If the receive count exceeds the retry limit, send the message to the DLQ and hand the booking to a reconciliation job that checks the payment provider by idempotency key before changing seat state.

## 4. Edge cases

### Server crashes after SQS publish but before API responds
The booking already exists in the database with `pending` status, and the message is already on SQS. The user may not receive the final response, but they do not get a duplicate booking because the API must be idempotent on `idempotencyKey`. A retry should return the same booking identity.

### Payment gateway returns a timeout
A timeout is ambiguous, so the worker must not mark the booking as failed immediately. It should retry with the same idempotency key and, before retrying too many times, query the gateway's status API if one exists. If the charge is still unresolved after the retry window, the message goes to the DLQ and reconciliation decides whether the payment actually completed.

## 5. SQS configuration

Visibility timeout:
- 120 seconds

Why:
- It is comfortably above the expected payment call plus DB confirmation path.
- It prevents duplicate delivery while a worker is still in the middle of a legitimate payment attempt.
- It is short enough that a stuck worker does not block the queue for too long.

Max receive count before DLQ:
- 5

Why:
- It allows transient gateway failures to recover.
- It keeps the retry window bounded so the system does not wait forever on a dead payment path.
- After five failed receives, the booking needs reconciliation rather than blind retries.
