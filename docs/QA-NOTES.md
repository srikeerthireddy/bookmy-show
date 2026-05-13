# Panel QA Notes

These are prepared notes for the five live-roast questions listed in the Part B prompt. If the panel wording differs, replace the answers below with the actual session notes.

## Q1: What happens if Redis crashes mid-lock?

**Answer notes:**
Redis is only the fast contention layer, not the correctness boundary. If Redis crashes after a lock is acquired, the lock TTL bounds the damage and the API falls back to Postgres as the source of truth. The final seat transition still happens in the database, so Redis loss can increase retries and latency but should not create double-bookings.

**Gap:**
The design still needs operational handling for a brief spike in retries while Redis recovers.

## Q2: At what concurrent users does the DB become the bottleneck?

**Answer notes:**
Using the pool math from [docs/CONCURRENCY.md](docs/CONCURRENCY.md), the synchronous path exhausts a 500-connection pool at about 2,841 RPS for the stated mix. The noon burst can reach about 8,333 RPS if 500,000 users arrive in 60 seconds, so the DB becomes the bottleneck well before raw CPU. That is why the design uses Redis for hot contention and SQS for async payment work.

**Gap:**
The exact limit depends on the real payment ratio and average gateway latency, so the 2,841 RPS number is a planning estimate, not a hard ceiling.

## Q3: What stops one user from holding 200 seats?

**Answer notes:**
The post-roast update adds a Redis counter, `holds:{userId}:count`, with a max of 8 active holds per user. The API increments the counter before creating a hold and decrements it on release, expiry, or booking confirmation. That caps abusive parallel holds from one account without changing the seat-lock model.

**Gap:**
It does not stop a coordinated botnet or many accounts acting together.

## Q4: Your auto-scaling spikes your bill to $3,200 this month - what's your plan?

**Answer notes:**
The design response is a publish circuit breaker around SQS plus CloudWatch alarms on queue depth and publish errors. If the async path degrades for more than 60 seconds, the API falls back to a synchronous payment attempt for low-volume traffic instead of letting retries pile up and burn cost. That preserves checkout availability while keeping the failure mode visible.

**Gap:**
The fallback reduces damage, but it does not eliminate the underlying cost of a real traffic spike.

## Q5: Why not PostgreSQL row locking instead of Redis?

**Answer notes:**
PostgreSQL row locking is correct, but the measured connection-pool math shows it becomes the bottleneck around 2,841 RPS on the stated mix, which is far below the 8,333 RPS noon burst. Redis `SETNX` handles hot contention faster and reduces how often the booking path has to sit on database connections. PostgreSQL still remains the final source of truth for the seat transition.

**Gap:**
Redis is not the correctness boundary, so this design still depends on careful TTLs and fallback behavior if Redis is unstable.