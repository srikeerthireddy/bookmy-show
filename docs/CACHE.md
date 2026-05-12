# Cache Design

## Cache policy

The cache uses a cache-aside pattern with delete-on-invalidation. Postgres remains the source of truth; Redis stores derived or read-heavy views. When something changes, we delete the key and let the next read repopulate it.

That approach is safer than write-through for this workload because most cached values are derived from seat state, and partial in-place updates would be harder to reason about under high contention.

## What to cache

### 1. Event details

Key:

`event:{event_id}:details`

Contents:
- name
- venue_id
- venue name
- city
- start_time
- status
- total_seat_count

TTL:
- 300 seconds

Why this TTL:
- Event metadata changes infrequently compared with seat status.
- Five minutes keeps the browse path hot without allowing stale cancellation or status data to linger too long.

Invalidation trigger:
- Event name, start time, status, or venue reference changes.
- Venue name or city changes if those values are embedded in the event payload.

Invalidation mode:
- Event-driven delete plus TTL fallback.

### 2. Seat availability count per event and category

Key:

`availability:{event_id}:{category}`

Contents:
- available_count
- held_count
- booked_count

TTL:
- 5 seconds

Why this TTL:
- Availability changes constantly during a sale and stale numbers hurt trust.
- Thirty seconds is too stale for a live ticket drop.
- Five minutes is useless for hot inventory.

Invalidation trigger:
- Any seat in that event and category changes status.
- A hold expires and the sweeper converts it back to available or the worker books it.
- Event status changes to sold_out or cancelled.

Invalidation mode:
- Event-driven delete plus short TTL.

### 3. Static seat map layout

Key:

`seatmap:{event_id}`

Contents:
- section
- row_label
- seat_number
- category
- base_price
- coordinates or layout metadata for the UI

TTL:
- 86,400 seconds

Why this TTL:
- The seat map layout is mostly static and can be cached aggressively.
- A one-day TTL is acceptable because layout changes are rare and will still be caught by invalidation.

Invalidation trigger:
- Any admin change to the seat layout, section mapping, or category mapping.
- Event reconfiguration that changes the renderable map.

Invalidation mode:
- Event-driven delete plus long TTL.

## What we explicitly will not cache

Individual seat status should not be cached as the primary decision source.

Why:
- It changes too often during a sale.
- It is the exact data that must stay correct to avoid double-bookings.
- A cached seat status can drift from the database and create a false sense of safety.

The database row is the truth for seat availability; Redis only stores derived views.

## Invalidation strategy

We delete the cache key on invalidation, then let the next read rebuild it.

Pseudocode:

```text
When seat status changes:
  1. Update the seat row in Postgres.
  2. Commit the transaction.
  3. Delete Redis key: availability:{event_id}:{category}
  4. Delete Redis key: event:{event_id}:details if the event status changed.
  5. The next read queries Postgres and repopulates Redis with a fresh TTL.
```

Why delete instead of update in place:
- Deletes are simpler and less error-prone.
- The next reader always gets a clean rebuild from the source of truth.
- It avoids partial writes across multiple cache keys.
