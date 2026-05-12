# Concurrency Strategy

## Constraint-driven math

The practical bottleneck is not raw CPU. It is database connections held open while a booking path waits on payment or seat validation.

Formula:

connections held = (% non-payment RPS x avg_query_time_s) + (% payment RPS x payment_hold_time_s)

With the stated mix:

connections held = (0.80 x RPS x 0.02) + (0.20 x RPS x 0.8)
connections held = 0.016 x RPS + 0.16 x RPS
connections held = 0.176 x RPS

With max_connections = 500 via PgBouncer:

500 = 0.176 x RPS
RPS = 2840.9

So the pool starts to exhaust at about 2,841 RPS for this traffic mix.

If the noon burst really turns into booking-related calls from all 500,000 users in 60 seconds, that is 8,333 RPS. At that level, a synchronous booking path would overwhelm a 500-connection pool long before the hardware is full.

## Option A: PostgreSQL SELECT FOR UPDATE

### How it prevents double-booking
Postgres is the final arbiter for seat ownership. The booking transaction locks every selected seat row before it changes state.

Example transaction:

```sql
BEGIN;

SELECT id, status, held_until, held_by, version
FROM seats
WHERE event_id = $1
  AND id = ANY($2)
ORDER BY id
FOR UPDATE;

-- validate that every requested seat is available and not expired
UPDATE seats
SET status = 'held',
    held_until = NOW() + INTERVAL '10 minutes',
    held_by = $user_id,
    version = version + 1,
    updated_at = NOW()
WHERE id = ANY($2)
  AND status = 'available';

INSERT INTO bookings (id, user_id, event_id, status, total_amount, idempotency_key)
VALUES ($booking_id, $user_id, $event_id, 'pending', $total_amount, $idempotency_key);

INSERT INTO booking_seats (booking_id, seat_id, seat_price)
SELECT $booking_id, id, price
FROM seats
WHERE id = ANY($2);

COMMIT;
```

The key guarantee is that only one transaction can lock and transition a seat row at a time. If a second request tries the same seat, it waits or fails instead of creating a duplicate confirmed booking.

### Hard limit
The hard limit is the connection pool. Once seat reservation and payment work collectively hold more than about 500 connections, the system queues behind the pool and latency climbs sharply.

Using the given workload mix, the pool exhausts at about 2,841 RPS. That is well below the theoretical 8,333 RPS noon burst, so pure synchronous booking over Postgres is not enough.

### Deadlock risk with multi-seat bookings
Multi-seat reservations can deadlock if two transactions lock the same seats in different orders.

Mitigation:
- Always sort seat IDs before locking.
- Lock rows in that deterministic order.
- Keep the transaction short.
- Fail fast with a lock timeout instead of waiting indefinitely.

## Option B: Redis SETNX distributed lock

### How it prevents double-booking
The lock key should be specific to the seat and the event:

seat_lock:{event_id}:{seat_id}

Acquire the lock with SET NX and a random token, then release it only if the stored token still matches. Use a Lua script for compare-and-delete so the release is atomic.

Example release script:

```lua
if redis.call('get', KEYS[1]) == ARGV[1] then
  return redis.call('del', KEYS[1])
end
return 0
```

### TTL choice
A 45-second TTL is long enough to cover a normal seat-selection and checkout attempt, but short enough to free abandoned holds reasonably quickly.

Too short:
- The lock can expire while the user is still in checkout, which reopens the seat early.

Too long:
- Abandoned carts block inventory for too long and reduce conversion.

### What happens if Redis fails mid-lock?
Redis alone cannot be the correctness boundary. If Redis loses the lock state before the booking is committed, another request can acquire the same lock and proceed. That is why Redis should not be the only protection for a no-double-booking promise.

## Chosen strategy: Hybrid

We choose a hybrid strategy because Postgres gives us the non-negotiable correctness guarantee, while Redis absorbs hot contention and reduces how often we hit the database on rejected or stale requests.

Why this choice:
- The pure-Postgres path hits pool exhaustion at about 2,841 RPS on the stated mix.
- The pure-Redis path is not safe enough as the only correctness layer because lock loss, expiry, or failover can reopen seats.
- The $2,000/month budget can support a small Redis tier plus Postgres and PgBouncer, which is still cheaper than trying to scale the database connection count blindly.

What this strategy cannot handle:
- A Redis outage removes the fast-fail layer and pushes more traffic back onto Postgres.
- A Postgres outage still stops sales entirely, because correctness lives there.
- A very long checkout flow can outlive a Redis lock TTL unless we renew or release it deliberately.

When we would switch:
- If traffic stays comfortably below roughly 2,000 to 2,500 RPS and the team wants the simplest possible stack, we would drop Redis and run pure Postgres locking.
- If the hot reservation path grows beyond what the pool can sustain, we would keep the hybrid approach and move even more admission control into Redis before the final Postgres commit.
