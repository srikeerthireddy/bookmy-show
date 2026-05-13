# Design Decisions

## Decision: Hybrid seat reservation strategy

**Context:** The noon burst can reach roughly 8,333 RPS, while the synchronous booking path starts exhausting a 500-connection pool at about 2,841 RPS on the stated traffic mix. We also cannot tolerate double-booking.

**Options considered:**
1. PostgreSQL `SELECT FOR UPDATE` only - correct, but it pushes every hot reservation through the same connection pool and becomes the bottleneck at the measured 2,841 RPS limit.
2. Redis `SETNX` locks only - fast, but Redis expiry, failover, or crash recovery cannot be the correctness boundary for seat ownership.
3. Hybrid Redis admission + PostgreSQL commit - chosen because Redis absorbs hot contention and stale requests while PostgreSQL makes the final seat transition.

**Why chosen:** The hybrid path is the only option that meets both constraints at once: it preserves correctness in PostgreSQL and still leaves room for hot-booking traffic on a budget that supports only a small Redis tier, PgBouncer, and a modest Postgres footprint.

**Tradeoffs accepted:** If Redis is unavailable, more contention falls back to Postgres and latency rises. The flow is more complex than a single locking primitive, and long checkout flows can outlive a short lock TTL if we do not renew or release it carefully.

**Revision trigger:** If traffic stays well below about 2,000 to 2,500 RPS, or if Redis becomes operationally noisy relative to its benefit, I would simplify to PostgreSQL locking only. If traffic grows beyond the current pool budget, I would add stronger Redis admission control before the final database commit.

## Decision: Cache invalidation with delete-on-change plus TTL fallback

**Context:** Event metadata changes infrequently, but seat availability changes constantly during a sale. Cached seat state is derived data, and stale availability hurts trust.

**Options considered:**
1. TTL-only invalidation - simple, but it leaves stale availability counts visible until expiry and makes sale-time data drift too easy to see.
2. Event-driven delete plus TTL fallback - chosen because it deletes affected keys after the Postgres commit and still lets the cache self-heal on the next read.

**Why chosen:** Delete-on-invalidation is safer than trying to patch multiple cached views in place, and the fallback TTLs keep the browse path hot without making Redis the source of truth.

**Tradeoffs accepted:** Hot-key invalidation can create short stampedes on the next read, and the design accepts eventual consistency for derived views such as availability counts.

**Revision trigger:** If invalidation churn or cache stampedes become the dominant latency source, I would add refresh-ahead or a fan-out mechanism for the hottest keys.

## Decision: UUID booking identifiers

**Context:** Booking records are created before payment completes, passed through SQS, retried by workers, and referenced across services. The identifier must be safe to generate at the edge and difficult to guess.

**Options considered:**
1. `SERIAL` or `BIGSERIAL` - simple, but it creates a sequence dependency, leaks ordering information, and is a poor fit for edge creation plus retries.
2. UUID - chosen because it is generated locally, works cleanly in queue messages, and avoids central sequence hot spots.

**Why chosen:** UUIDs let the API create the booking immediately without waiting on shared sequence allocation, and they fit the async queue flow without exposing booking volume or making identifiers easy to predict.

**Tradeoffs accepted:** UUID indexes are larger and slightly less locality-friendly than sequential IDs, so storage and index maintenance cost more than a narrow serial key.

**Revision trigger:** If bookings become storage-bound or index bloat becomes a material operational problem, I would revisit the public identifier strategy and consider a surrogate internal key plus a separate public token.

## Decision: SQS visibility timeout at 120 seconds

**Context:** The payment worker must cover queue receive, payment gateway latency, and database confirmation without letting a stuck worker block the queue for too long.

**Options considered:**
1. Short timeout such as 30 to 60 seconds - easier to requeue failures, but too aggressive for legitimate payment attempts and confirmation writes.
2. 120 seconds - chosen because it is comfortably above the expected payment plus database-confirmation path.

**Why chosen:** 120 seconds balances the two failure modes we care about: it is long enough to avoid duplicate delivery while a worker is still doing legitimate work, and short enough that a stuck worker does not hide the message indefinitely. The retry window stays bounded because the queue also dead-letters after 5 receives.

**Tradeoffs accepted:** Failure detection is slower than with a very short timeout, and a worker that stalls past 120 seconds can still lead to duplicate processing if idempotency is broken elsewhere.

**Revision trigger:** If payment latency distributions rise toward the current timeout or the worker confirmation path starts to approach 120 seconds at p95, I would raise the timeout and split the confirmation step further. If duplicates become common, I would shorten it only after strengthening idempotency and worker heartbeats.
