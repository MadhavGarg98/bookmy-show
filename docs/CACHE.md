# Cache Design

This document details the Redis caching strategy used in the ShowTime architecture to minimize database load while maintaining acceptable data freshness for the user.

## What to Cache

### 1. Event Details
- **Redis Key Format:** `event:{event_id}`
- **TTL:** `3600` seconds (1 hour)
- **Justification:** Event details (name, venue location, scheduled time) are highly static. They rarely change once an event goes on sale. A 1-hour TTL acts as a safety net against stale data if invalidation misses, while keeping DB hits to near zero during peak sale hours.
- **Invalidation Trigger:** Whenever the `events` table is updated via the Admin panel.
- **Strategy:** Event-driven invalidation (delete key on DB update) + TTL expiry.

### 2. Seat Availability Counts
- **Redis Key Format:** `availability:{event_id}:{category}`
- **TTL:** `30` seconds
- **Justification:** During a high-traffic sale, availability counts change thousands of times per second. Querying PostgreSQL with `COUNT(*)` continuously will immediately kill the database. A 30-second TTL allows the API to serve slightly stale but highly performant counts. Showing "42 seats left" instead of "38" is an acceptable UX tradeoff compared to the site crashing.
- **Invalidation Trigger:** Whenever a seat status changes from 'available' to 'held', or 'held' to 'available'.
- **Strategy:** Targeted Invalidation. A 30s pure TTL means users could look at bad data for up to 30 seconds after seats are sold out. By actively invalidating on status changes, the cache repopulates faster during critical sell-outs.

### 3. Static Seat Map Layout
- **Redis Key Format:** `seatmap:{event_id}`
- **TTL:** `86400` seconds (24 hours)
- **Justification:** The physical layout of a venue (sections, rows, graphical coordinates) does not change during an event. This is a heavy JSON blob that should be cached indefinitely.
- **Invalidation Trigger:** Only upon event cancellation or rare physical venue restructuring.
- **Strategy:** Purely event-driven invalidation (Admin action).

## What NEVER to Cache
🚨 **Individual Seat Status (`seat:{seat_id}:status`)**
We will explicitly **NOT** cache whether seat `A-12` is available or booked. Individual seat status must always be verified against the source of truth (the Redis SETNX lock state for holds, and PostgreSQL for final bookings). Caching this creates a dangerous window where two users see a cached "available" status, click book simultaneously, and hit the concurrency locks, resulting in higher conflict rates and poor UX. Individual seat status is evaluated at the exact moment of checkout.

## The Invalidation Strategy

We utilize the **Cache-Aside** pattern with targeted deletion, rather than in-place updates.

### Why Delete instead of Update (Write-Through)?
Updating a complex JSON object or incrementing/decrementing counts in Redis introduces race conditions between concurrent worker nodes. It is safer to simply delete the key and let the next incoming read request re-fetch the accurate, atomic state from PostgreSQL.

### Code-Level Logic Flow

```javascript
// Example: Seat A-12 moves from 'available' to 'held'

// 1. Update the Source of Truth (Database)
await db.query(
  'UPDATE seats SET status=$1, version=version+1 WHERE id=$2', 
  ['held', seatId]
);

// 2. Invalidate the associated availability count in Cache
// We delete the key. The next user who views the UI will trigger a cache miss,
// forcing the API to run a COUNT(*) query and repopulate the fresh 30s cache.
await redis.del(`availability:${eventId}:${category}`);
```
