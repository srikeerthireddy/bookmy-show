# ShowTime / BookMyShow Competitor

This repository captures the Part A foundation for a high-stakes ticketing system: the database schema, concurrency strategy, cache layer, and async order flow.

## Constraints

### 1. 5 lakh users at noon
If 500,000 users hit the system inside 60 seconds and each user makes one booking-related request, the peak load is about 8,333 RPS. That means the browse path must be cache-heavy and the booking path must avoid holding database connections longer than necessary.

### 2. Zero acceptable double-bookings
A double-booking means two confirmed booking rows end up owning the same seat. The only acceptable design is one that makes the seat state transition atomic in the database, with the database as the final source of truth.

### 3. $2,000/month AWS budget
The budget is enough for a small PostgreSQL primary, PgBouncer, a Redis cache, SQS, and an ALB. It does not buy a wide, multi-region consensus system. The design must therefore stay simple, keep hot state in Postgres, and push everything else to cache or async workers.

### What changes if the budget drops to $500/month
The first things to remove would be Redis as a dedicated layer and any nonessential background processing capacity. The system would fall back to a more Postgres-centric design with tighter cache TTLs and lower peak throughput expectations.

## Deliverables

- [SCHEMA.md](docs/SCHEMA.md)
- [CONCURRENCY.md](docs/CONCURRENCY.md)
- [CACHE.md](docs/CACHE.md)
- [QUEUE.md](docs/QUEUE.md)

## Part B placeholders

These files are created now and will be filled later for the diagram and defense round.

- [ARCHITECTURE.md](docs/ARCHITECTURE.md)
- [DESIGN-DECISIONS.md](docs/DESIGN-DECISIONS.md)
- [DESIGN-UPDATES.md](docs/DESIGN-UPDATES.md)
