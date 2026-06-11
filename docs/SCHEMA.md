# Database Schema Design

## PostgreSQL Schema (v14+)

```sql
-- Venues table
CREATE TABLE venues (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    city VARCHAR(100) NOT NULL,
    address TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_venues_city ON venues(city);

-- Events table
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    venue_id UUID NOT NULL REFERENCES venues(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    event_time TIMESTAMPTZ NOT NULL,
    booking_opens_at TIMESTAMPTZ NOT NULL,
    booking_closes_at TIMESTAMPTZ NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CHECK (booking_opens_at < booking_closes_at),
    CHECK (booking_closes_at <= event_time)
);

CREATE INDEX idx_events_venue ON events(venue_id);
CREATE INDEX idx_events_booking_window ON events(booking_opens_at, booking_closes_at);

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Seats table (pre-populated per venue)
CREATE TABLE seats (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    seat_number VARCHAR(10) NOT NULL,
    row_label VARCHAR(5) NOT NULL,
    section VARCHAR(50) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price >= 0),
    status VARCHAR(20) NOT NULL DEFAULT 'available',
    version INT NOT NULL DEFAULT 0,
    held_until TIMESTAMPTZ,
    CHECK (status IN ('available', 'held', 'booked')),
    UNIQUE(event_id, seat_number)
);

-- Critical indexes for concurrency
CREATE INDEX idx_seats_event_status ON seats(event_id, status);
CREATE INDEX idx_seats_held_until ON seats(held_until) WHERE held_until IS NOT NULL;

-- Partial index for rapid seat selection queries
CREATE INDEX idx_seats_available ON seats(event_id) WHERE status = 'available';

-- Bookings table
CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    event_id UUID NOT NULL REFERENCES events(id) ON DELETE CASCADE,
    total_amount DECIMAL(10,2) NOT NULL CHECK (total_amount >= 0),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    payment_id VARCHAR(255),
    idempotency_key VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMPTZ NOT NULL,
    completed_at TIMESTAMPTZ,
    CHECK (status IN ('pending', 'confirmed', 'cancelled', 'expired'))
);

CREATE INDEX idx_bookings_user ON bookings(user_id);
CREATE INDEX idx_bookings_status ON bookings(status);

-- Partial index for monitoring pending bookings
CREATE INDEX idx_bookings_pending ON bookings(expires_at) WHERE status = 'pending';

-- Booking-Seat junction table
CREATE TABLE booking_seats (
    booking_id UUID NOT NULL REFERENCES bookings(id) ON DELETE CASCADE,
    seat_id UUID NOT NULL REFERENCES seats(id) ON DELETE CASCADE,
    price_paid DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (booking_id, seat_id)
);

CREATE INDEX idx_booking_seats_seat ON booking_seats(seat_id);
```

## Design Commentary

### UUID vs SERIAL
- **Choice:** UUID (gen_random_uuid())
- **Rationale:** Distributed-friendly, prevents enumeration attacks, merge-safe for sharding
- **Trade-off:** 16 bytes vs 8 bytes (acceptable for scale)

### Version Column (Optimistic Locking)
- Tracks seat update count
- Enables optimistic concurrency control
- Alternative to row-level locks for high contention

### held_until Usage
- Seats transition: `available → held (with TTL) → booked`
- Prevents indefinite locks from abandoned carts
- Background job expires held seats after TTL
- Index enables fast cleanup queries

### Partial Indexes
- `idx_seats_available`: Only indexes available seats (reduces index size by 90% post-sale)
- `idx_bookings_pending`: Fast TTL expiry checks without scanning confirmed bookings
- Benefit: Faster queries + reduced index maintenance overhead

### Foreign Key ON DELETE Behavior
- `CASCADE`: Deleting event removes seats/bookings (admin cleanup)
- Prevents orphaned records
- Trade-off: Use soft deletes in production to preserve history
