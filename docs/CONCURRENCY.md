# Concurrency Strategy

This document outlines the concurrency strategies considered for the ShowTime ticketing backend and justifies the chosen approach to ensure zero double-bookings while surviving 5 Lakh concurrent users at launch.

## Option A: PostgreSQL Row-Level Locking (`SELECT FOR UPDATE`)

### How it Prevents Double-Booking
When a user attempts to book a seat, the application opens a database transaction and executes a `SELECT ... FOR UPDATE` on the specific seat row. This places a pessimistic row-level lock on the seat. If another transaction tries to read or update that same seat, it is forced to wait until the first transaction either commits or rolls back.

```sql
BEGIN;
SELECT id, status FROM seats WHERE id = ANY($1::int[]) AND status = 'available' FOR UPDATE;
UPDATE seats SET status = 'held', held_until = NOW() + INTERVAL '10 minutes', held_by = $2 WHERE id = ANY($1::int[]);
INSERT INTO bookings ...
COMMIT;
```

### The Capacity Hard Limit (Pool Exhaustion Math)
With a budget allowing only a primary RDS instance, we rely on PgBouncer to manage a connection pool of `max_connections = 500`.

**Formula:**
`Connections held = (% non-payment RPS × avg_query_time_s) + (% payment RPS × payment_hold_time_s)`

Assume 20% of traffic reaches the payment hold phase (taking 800ms due to external latency or complex logic), and 80% is regular fast queries (20ms).
- Max Connections = 500
- 500 = (0.80 × RPS × 0.02s) + (0.20 × RPS × 0.80s)
- 500 = RPS × (0.016) + RPS × (0.16)
- 500 = RPS × 0.176
- **RPS Limit ≈ 2,840 RPS**

At around 2,840 RPS, all 500 connections in the pool are simultaneously locked. At 5 Lakh users (~8,333+ RPS burst), the pool will exhaust instantly. 

### Deadlock Risk
If User 1 tries to book seats `[10, 11]` and User 2 simultaneously tries to book seats `[11, 10]`, User 1 locks 10, User 2 locks 11. User 1 waits for 11, and User 2 waits for 10. This is a deadlock.
**Mitigation:** The application must strictly sort the `seat_id` array in ascending order before passing it to the database, ensuring all concurrent transactions attempt to acquire locks in the exact same sequence.

---

## Option B: Redis Distributed Lock (`SETNX`)

### How it Prevents Double-Booking
Instead of locking in PostgreSQL, the application attempts to set an atomic flag in Redis. `SETNX` (Set if Not eXists) ensures only one API request can successfully create the key. The winner proceeds to update the database; losers immediately fail and return "Seat unavailable" without touching the DB.

### Handling Redis Failures
If Redis crashes after acquiring a lock but before the database is updated, the lock eventually expires via its Time-To-Live (TTL). However, if Redis fails *completely* (node down), the system loses its locking capability. If we rely purely on Redis, this would cause double-bookings. Therefore, Redis must be backed by DB-level optimistic concurrency (the `version` column).

### Lock TTL
For the initial lock to protect the database update (status -> 'held'), a **30-second TTL** is appropriate. It provides enough time for the API to run the database query, but ensures that if the API server crashes, the lock is released quickly. Once the seat is marked as 'held' in PostgreSQL, the Redis lock is deleted via a Lua script. (The actual *hold* duration given to the user for payment is 10 minutes, managed by PostgreSQL).

### Lock Key Format
`seat_lock:{event_id}:{seat_id}`
Including the `event_id` provides natural namespace separation and makes it easier to flush locks for a specific event if necessary.

---

## Chosen Strategy: Hybrid (Redis SETNX + Postgres Optimistic Locking)

**We choose a Hybrid approach because:**
1. **Capacity Constraints:** Pure Postgres `SELECT FOR UPDATE` fails catastrophically around 2,840 RPS, which is insufficient for the 5 Lakh user burst.
2. **Budget Constraints:** Our $2,000 budget prevents us from scaling PostgreSQL horizontally to absorb the lock contention, but it *does* allow for a 3-node ElastiCache Redis cluster capable of >100,000 sub-millisecond ops/sec.

**How it works:**
1. When a user clicks "Book", the API first acquires a Redis `SETNX` lock (`seat_lock:{event_id}:{seat_id}`, TTL: 30s).
2. Only if the Redis lock is acquired does the API query PostgreSQL.
3. The API updates the seat to 'held' in PostgreSQL using optimistic locking (`UPDATE seats SET status='held', version=version+1 WHERE id=$1 AND version=$2`).
4. The Redis lock is released.

**Tradeoffs & Limitations:**
This strategy is highly performant but complex. It cannot handle severe network partitions between the API and Redis seamlessly. If Redis goes down, we fallback entirely to PostgreSQL's optimistic locking, which will result in higher database load and potential `update conflicts` returned to users, but *guarantees zero double-bookings*.
