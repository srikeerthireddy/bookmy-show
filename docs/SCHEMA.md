# PostgreSQL Schema

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE venues (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    city TEXT NOT NULL,
    capacity INTEGER NOT NULL CHECK (capacity > 0)
);

CREATE TABLE events (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL,
    venue_id BIGINT NOT NULL REFERENCES venues(id) ON DELETE RESTRICT,
    start_time TIMESTAMPTZ NOT NULL,
    status TEXT NOT NULL CHECK (status IN ('upcoming', 'on_sale', 'sold_out', 'cancelled')),
    total_seat_count INTEGER NOT NULL CHECK (total_seat_count > 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_events_venue_start_time ON events (venue_id, start_time);
CREATE INDEX idx_events_status_start_time ON events (status, start_time);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email TEXT NOT NULL UNIQUE,
    phone TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_created_at ON users (created_at DESC);

CREATE TABLE seats (
    id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    section TEXT NOT NULL,
    row_label TEXT NOT NULL,
    seat_number INTEGER NOT NULL CHECK (seat_number > 0),
    price NUMERIC(10, 2) NOT NULL CHECK (price > 0),
    category TEXT NOT NULL CHECK (category IN ('VIP', 'General', 'Premium')),
    status TEXT NOT NULL CHECK (status IN ('available', 'held', 'booked')),
    held_until TIMESTAMPTZ,
    held_by UUID REFERENCES users(id) ON DELETE SET NULL,
    version INTEGER NOT NULL DEFAULT 0 CHECK (version >= 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT uq_seat_position UNIQUE (event_id, section, row_label, seat_number),
    CONSTRAINT chk_seat_hold_state CHECK (
        (status = 'available' AND held_until IS NULL AND held_by IS NULL)
        OR (status = 'held' AND held_until IS NOT NULL AND held_by IS NOT NULL)
        OR (status = 'booked')
    )
);

CREATE INDEX idx_seats_event_status ON seats (event_id, status);
CREATE INDEX idx_seats_event_category ON seats (event_id, category);
CREATE INDEX idx_seats_event_hold_expiry ON seats (event_id, held_until);

CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    event_id BIGINT NOT NULL REFERENCES events(id) ON DELETE RESTRICT,
    status TEXT NOT NULL CHECK (status IN ('pending', 'confirmed', 'failed', 'refunded')),
    total_amount NUMERIC(10, 2) NOT NULL CHECK (total_amount > 0),
    payment_reference TEXT,
    idempotency_key TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    confirmed_at TIMESTAMPTZ,
    failed_at TIMESTAMPTZ,
    refunded_at TIMESTAMPTZ,
    CONSTRAINT uq_bookings_idempotency UNIQUE (user_id, idempotency_key)
);

CREATE INDEX idx_bookings_user ON bookings (user_id, created_at DESC);
CREATE INDEX idx_bookings_unresolved ON bookings (event_id, created_at DESC) WHERE status IN ('pending', 'failed');
CREATE INDEX idx_bookings_payment_reference ON bookings (payment_reference) WHERE payment_reference IS NOT NULL;

CREATE TABLE booking_seats (
    booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    seat_id BIGINT NOT NULL REFERENCES seats(id) ON DELETE RESTRICT,
    seat_price NUMERIC(10, 2) NOT NULL CHECK (seat_price > 0),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (booking_id, seat_id),
    UNIQUE (seat_id)
);

CREATE INDEX idx_booking_seats_booking ON booking_seats (booking_id);
```

## Commentary

### Why UUID for `bookings.id` instead of SERIAL?
Bookings are customer-facing objects that may need to be created before the payment finishes, passed through a queue, and safely referenced across retries. UUIDs are safe to generate at the edge, avoid sequence hot spots, and make it harder to guess the next booking identifier.

### Why does `seats` have a `version` column?
The version column gives us optimistic locking for stale updates and a clean way to detect if a seat row changed between read and write. It is a safety net for concurrent flows, reconciliation jobs, and any update path that should fail fast when it is operating on stale seat state.

### Why `held_until` instead of just holding seats at the application level?
A database column makes the hold durable, queryable, and recoverable after crashes. If the application dies after reserving a seat, the expiry timestamp still exists in Postgres, so a sweeper or worker can safely release the seat without relying on in-memory state.

### Why a partial index on `bookings.status`?
The hot set is tiny: only unresolved bookings need fast access during checkout, retries, and reconciliation. A partial index keeps the active rows compact and avoids paying index maintenance cost on old confirmed or refunded history.
