# Full System Architecture

```text
									  +--------------------------------------+
									  | Users / Browsers                     |
									  +------------------+-------------------+
														 |
														 | HTTPS requests for static assets,
														 | cached event pages, and API calls
														 v
		+----------------------------------------------------------------------------------------+
		| CloudFront CDN (CACHE.md: browse path must be cache-heavy)                            |
		| - serves static assets and cached event pages                                          |
		| - cache hit: returns edge-cached HTML, CSS, JS, and event pages to users              |
		| - cache miss / API pass-through: forwards uncached page requests and API calls        |
		+----------------------------+------------------------------------+----------------------+
									 |                                    |
									 | cache hit                          | cache miss / API pass-through
									 |                                    v
									 |                    +-----------------------------------------------+
									 |                    | Application Load Balancer (README.md:         |
									 |                    | 8,333 RPS noon burst needs a thin origin tier) |
									 |                    | - SSL termination                             |
									 |                    | - health checks every 15s                      |
									 |                    | - rate limit rule on anonymous browse traffic  |
									 |                    +-------------------+---------------------------+
									 |                                        |
									 |                                        | HTTPS to Node.js API auto-scale group
									 |                                        v
									 |   +---------------------------------------------------------------------------------------------+
									 |   | Node.js API servers, auto-scale group (QUEUE.md: reserve seats, then enqueue payment work) |
									 |   | - read Redis cache keys with TTL                                                            |
									 |   | - read Redis seat locks acquired with SETNX and lock TTL                                    |
									 |   | - write booking, seat, and hold state to PostgreSQL                                         |
									 |   | - publish payment work to SQS                                                               |
									 |   +----------------------------+-------------------------------------------+----------------------+
									 |                                |                          |
									 |                                | cache read / seat-lock check             | write transactions    | publish payment message
									 |                                v                          v                      v
									 |   +-----------------------------------------------+        +-----------------------------------+   +--------------------------------------------------+
									 |   | Redis Cluster (CONCURRENCY.md: 500K-user      |        | PostgreSQL Primary                |   | SQS Payment Queue (QUEUE.md: async payment keeps |
									 |   | burst needs fast contention handling)         |        | (SCHEMA.md: writes only final     |   | the pool below exhaustion)                       |
									 |   | - cache keys with TTL                         |        | source of truth for seat state)   |   | message summary: bookingId, userId, eventId,     |
									 |   | - seat locks via SETNX with lock TTL          |        | - bookings                         |   | seatIds, amount, currency, paymentToken,         |
									 |   |                                               |        | - booking_seats                    |   | idempotencyKey                                   |
									 |   +---------------------------+-------------------+        | - seats                            |   | visibility timeout: 120s                         |
									 |                               |                            | - events                           |   | DLQ path after 5 receives                         |
									 |                               | cache miss -> read-through | - users                            |   +----------------------+---------------------------+
									 |                               v                            +-------------------+-------------------+                          |
									 |   +-----------------------------------------------+                            |                                          | message consumed
									 |   | PostgreSQL Read Replicas x2 (SCHEMA.md: browse |                            | final seat/booking commit                v
									 |   | reads should not hit the primary)              |                            |                          +------------------------------------------+
									 |   | - event details                                 |                            |                          | Payment Worker (ECS) (QUEUE.md: worker  |
									 |   | - seat availability counts                      |                            |                          | owns async confirmation flow)            |
									 |   | - seat map reads                                 |                            |                          | 1. read SQS                              |
									 |   +---------------------------+-------------------+                            |                          | 2. call payment gateway                  |
									 |                               |                                                |                          | 3. update DB                             |
									 |                               | browse read response                          |                          | 4. publish SNS                           |
									 |                               v                                                |                          | 5. delete message                        |
									 |                   +---------------------------------+                           |                          +--------------------+---------------------+
									 |                   | API response to browser         |                           |                                               |
									 |                   | cache miss filled from replica   |                           |                                               | payment confirmation event
									 |                   +---------------------------------+                           |                                               v
									 |                                                                                      |                             +-----------------------------------+
									 |                                                                                      |                             | SNS Topic -> SES Email + SMS      |
									 |                                                                                      |                             | (QUEUE.md: confirmation fan-out   |
									 |                                                                                      |                             | on successful payment)            |
									 |                                                                                      |                             +-----------------------------------+
									 |                                                                                      |
									 |                                                                                      | write confirmation / failure state
									 |                                                                                      v
									 |                                                                           +-------------------------------+
									 |                                                                           | Payment Gateway (external)    |
									 |                                                                           | called with idempotency key   |
									 |                                                                           +-------------------------------+
```

## Component annotations

Every box above is tied back to a Part A choice:

- CloudFront CDN -> CACHE.md: browse path must stay cache-heavy.
- Application Load Balancer -> README.md: the noon burst needs a thin edge/origin tier under budget.
- Node.js API servers -> QUEUE.md: the API must reserve seats quickly and push payment work out of band.
- Redis Cluster -> CONCURRENCY.md: Redis absorbs hot contention with SETNX locks and TTL-based cache keys.
- SQS Payment Queue -> QUEUE.md: synchronous payment would exhaust the pool at roughly 2,841 RPS.
- Payment Worker -> QUEUE.md: the worker owns payment confirmation, retries, and DLQ handling.
- PostgreSQL Primary -> SCHEMA.md: writes land here because the database is the final source of truth.
- PostgreSQL Read Replicas x2 -> SCHEMA.md: read-heavy browse queries should not contend with writes.
- SNS -> SES Email + SMS -> QUEUE.md: payment confirmation is the async fan-out trigger.
