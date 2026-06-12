# Design Decisions

## Decision: Concurrency Strategy
 
**Context:** Need to handle 500,000 concurrent users at launch without double-booking seats while remaining within a $2,000/month budget.
 
**Options considered:**
1. PostgreSQL Row-Level Locking (`SELECT FOR UPDATE`) - Considered for strong consistency, but not chosen because it exhausts a 500-connection DB pool at ~2,840 RPS.
2. Redis Distributed Lock (`SETNX`) + Postgres Optimistic Locking - Chosen approach. Redis acts as a fast initial barrier, while DB version column provides the final guarantee.
 
**Why chosen:** 
Pure PostgreSQL locking fails at 2,840 RPS. The hybrid approach leverages a 3-node ElastiCache Redis cluster capable of >100,000 sub-millisecond ops/sec, fitting within budget and scaling to meet the 5 Lakh user burst.
 
**Tradeoffs accepted:** 
Increased system complexity and partial reliance on Redis uptime. If Redis partitions or fails, throughput degrades severely as all load falls back to PostgreSQL optimistic locking, resulting in high update conflicts for users.
 
**Revision trigger:** 
If the budget increases significantly, allowing for distributed databases (like CockroachDB or TiDB) that can handle high-throughput row locking horizontally, we would remove Redis locks.

## Decision: Cache Invalidation Approach
 
**Context:** Seat availability counts change thousands of times per second during peak sales, which would overwhelm the database if queried directly.
 
**Options considered:**
1. Event-driven invalidation (Write-Through) - Considered for strong consistency, but not chosen because updating complex counts under heavy concurrency creates race conditions.
2. Cache-Aside with Targeted Deletion + TTL - Chosen approach. We delete keys on status changes and rely on a 30-second TTL as a fallback.
 
**Why chosen:** 
It avoids race conditions by forcing the next read to fetch atomic state from PostgreSQL. The 30s TTL acts as a safety net. This is highly performant and keeps DB hits near zero during peaks.
 
**Tradeoffs accepted:** 
Users may see slightly stale availability data (e.g., showing 42 seats instead of 38) for up to 30 seconds if targeted deletion misses.
 
**Revision trigger:** 
If product requirements dictate that "Seats Left" must be 100% accurate in real-time on the UI (no stale reads allowed), we would need a WebSocket-based push architecture.

## Decision: UUID vs SERIAL for booking IDs
 
**Context:** We need a primary key strategy for the `bookings` table that supports high volume and security.
 
**Options considered:**
1. `SERIAL` (Auto-incrementing Integer) - Considered for simplicity and index performance, but not chosen because it exposes business metrics (competitors can guess total sales volume) and creates IDOR vulnerabilities.
2. `UUID v4` - Chosen approach. Cryptographically random and unguessable.
 
**Why chosen:** 
UUIDs hide overall volume, prevent sequential guessing attacks, and can be generated on the client or API server before database insertion, enabling safer idempotency.
 
**Tradeoffs accepted:** 
UUIDs consume more storage (16 bytes vs 4 bytes) and can cause index fragmentation in PostgreSQL compared to sequential integers.
 
**Revision trigger:** 
If database storage costs or insert performance degradation due to index bloat becomes a primary bottleneck, we might switch to a time-sortable format like ULID or Twitter Snowflake.

## Decision: SQS Visibility Timeout
 
**Context:** The payment processing pipeline is asynchronous to prevent API servers from holding database connections for 2,000ms. We need to configure the queue safely.
 
**Options considered:**
1. Standard visibility timeout (e.g., 5 seconds) - Not chosen because payment gateway calls can occasionally take up to 10-15 seconds before timing out.
2. 30 Seconds visibility timeout - Chosen approach.
 
**Why chosen:** 
Payment gateways usually respond in <2 seconds, but timeouts can take up to 10 seconds. 30 seconds guarantees the original worker has either completed or completely crashed before a second worker attempts the same message, preventing double-processing.
 
**Tradeoffs accepted:** 
If a worker crashes immediately after picking up a message, that user's booking confirmation is delayed by at least 30 seconds until the message reappears in the queue.
 
**Revision trigger:** 
If we switch to a faster internal payment ledger or if our payment gateway provider guarantees a maximum latency of 3 seconds, we could reduce the timeout to 10 seconds to recover from worker crashes faster.
