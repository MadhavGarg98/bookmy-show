# Database Schema Design

This document contains the PostgreSQL Data Definition Language (DDL) for the ShowTime ticketing backend, designed to handle extreme concurrency.

## Data Definition Language (DDL)

```sql
CREATE TABLE venues (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    city VARCHAR(255) NOT NULL,
    capacity INT NOT NULL CHECK (capacity > 0)
);

CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    venue_id INT NOT NULL,
    start_time TIMESTAMP WITH TIME ZONE NOT NULL,
    status VARCHAR(50) NOT NULL CHECK (status IN ('upcoming', 'on_sale', 'sold_out', 'cancelled')),
    total_seat_count INT NOT NULL CHECK (total_seat_count > 0),
    FOREIGN KEY (venue_id) REFERENCES venues(id) ON DELETE RESTRICT
);

CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL
);

CREATE TABLE seats (
    id SERIAL PRIMARY KEY,
    event_id INT NOT NULL,
    section VARCHAR(100) NOT NULL,
    row_num VARCHAR(10) NOT NULL,
    seat_number VARCHAR(10) NOT NULL,
    price DECIMAL(10, 2) NOT NULL CHECK (price > 0),
    category VARCHAR(50) NOT NULL CHECK (category IN ('VIP', 'General', 'Premium')),
    status VARCHAR(50) NOT NULL CHECK (status IN ('available', 'held', 'booked')),
    held_until TIMESTAMP WITH TIME ZONE,
    held_by UUID,
    version INT NOT NULL DEFAULT 0,
    FOREIGN KEY (event_id) REFERENCES events(id) ON DELETE CASCADE,
    FOREIGN KEY (held_by) REFERENCES users(id) ON DELETE SET NULL,
    UNIQUE (event_id, section, row_num, seat_number)
);

CREATE TABLE bookings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    event_id INT NOT NULL,
    status VARCHAR(50) NOT NULL CHECK (status IN ('pending', 'confirmed', 'failed', 'refunded')),
    total_amount DECIMAL(10, 2) NOT NULL CHECK (total_amount > 0),
    payment_reference VARCHAR(255),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE RESTRICT,
    FOREIGN KEY (event_id) REFERENCES events(id) ON DELETE RESTRICT
);

CREATE TABLE booking_seats (
    booking_id UUID NOT NULL,
    seat_id INT NOT NULL,
    price_at_booking DECIMAL(10, 2) NOT NULL CHECK (price_at_booking > 0),
    PRIMARY KEY (booking_id, seat_id),
    FOREIGN KEY (booking_id) REFERENCES bookings(id) ON DELETE CASCADE,
    FOREIGN KEY (seat_id) REFERENCES seats(id) ON DELETE RESTRICT
);

-- Critical Indexes
CREATE INDEX idx_seats_event_status ON seats(event_id, status);
CREATE INDEX idx_bookings_user ON bookings(user_id, created_at DESC);

-- Partial index for unresolved bookings
CREATE INDEX idx_bookings_unresolved ON bookings(status) WHERE status IN ('pending', 'failed');
```

## Schema Design Decisions

### Why UUID for `booking.id` instead of `SERIAL`?
Using `SERIAL` integers for bookings exposes business metrics and introduces a security vulnerability (Insecure Direct Object Reference). An attacker or competitor could easily iterate over integer IDs (`/bookings/1`, `/bookings/2`) to scrape all historical bookings or infer the total volume of sales. UUID v4 is cryptographically random, hiding overall volume and preventing sequential guessing. Furthermore, UUIDs can be generated on the client or API server *before* inserting into the database, enabling safer idempotency and retry logic.

### Why does `seats` have a `version` column?
The `version` column enables **Optimistic Concurrency Control**. In an extremely high-concurrency scenario (5 Lakh users), acquiring heavy row locks (`SELECT FOR UPDATE`) causes database connection pools to exhaust quickly. Instead, by keeping a version number, multiple requests can read the same seat state, and the first request to update the row will increment the version (`UPDATE seats SET ... version = version + 1 WHERE id = X AND version = Y`). Any subsequent attempt will fail to update (returning 0 rows affected) because the version no longer matches, preventing double-booking without holding expensive DB locks for long durations.

### Why `held_until` instead of just holding seats at the application level?
Application-level memory holds are volatile. If an API server crashes while a user is entering their payment details, the memory hold disappears, and another user might snatch the seat while the first user thinks they have it reserved. By storing `held_until` in the database, the hold is durable across API server nodes. Background workers or periodic sweepers can accurately release expired holds, and the payment worker reliably verifies ownership directly against the database source of truth.

### Why a partial index on `bookings.status`?
The majority of bookings (often 99%) quickly settle into final states: `confirmed` or `refunded`. These historical records are rarely queried in bulk. However, background workers that process async payments, clean up expired holds, or retry failed operations need to constantly poll for `pending` or `failed` bookings. A partial index (`WHERE status IN ('pending', 'failed')`) remains exceptionally small and entirely memory-resident, allowing the database to instantly identify the few active rows among millions of historical ones without a full table scan.
