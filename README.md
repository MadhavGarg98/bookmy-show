# ShowTime Architecture - Part A

Welcome to the ShowTime Backend Architecture repository! This document contains the initial constraint analysis for the BookMyShow scale simulation. Detailed documentation can be found in the `docs` directory.

## Constraints Analysis

### Constraint 1: 5 Lakh Concurrent Users at 12:00:00 Noon

**Peak RPS Calculation:**
If 5,000,000 users are active and attempting to perform an action (like refreshing the page or booking a ticket) within a 60-second window, the peak Requests Per Second (RPS) is approximately:
`500,000 users / 60 seconds ≈ 8,333 RPS`

At the exact stroke of 12:00:00, the burst traffic might even be significantly higher as all users tap "Book" simultaneously.
**What component hits its limit first?**
At ~8,300+ RPS, the **Database Connection Pool** will hit its limit first. PostgreSQL, even heavily tuned with PgBouncer, safely manages hundreds or low thousands of concurrent connections. If each request blocks a connection, the pool will exhaust almost immediately, queueing the rest and causing API timeouts.

### Constraint 2: Zero Acceptable Double-Bookings

**What does a double-booking mean technically?**
A double-booking occurs when two rows are inserted into the `booking_seats` table referencing the same `seat_id` for a specific event where the seat is expected to only be held by one user. Or, similarly, the `seats` table has its status overwritten by two concurrent transactions resulting in two users thinking they have successfully paid for the same seat.

**What mechanisms prevent this?**
To prevent this, we need strict locking mechanisms.
- **Pessimistic Locking (e.g., PostgreSQL `SELECT FOR UPDATE`):** Locks the row during the entire transaction, strictly ensuring no other transaction can read/modify the row until completion. But this holds connections longer.
- **Distributed Locking (e.g., Redis `SETNX`):** Acquires a lock in memory (Redis). This is faster and offloads contention from the database, but requires separate lock management and TTL tuning.
- **Optimistic Locking (e.g., via `version` columns):** Allows concurrent reads but checks if a record has been modified before updating. Fails the slower transaction cleanly without database-level blocking.

### Constraint 3: $2,000/Month AWS Budget

**What does $2,000 actually buy?**
A $2,000/month budget approximately affords a capable but limited infrastructure:
- 6x `t3.xlarge` EC2 instances for API servers (~$720/mo)
- 1x `db.r6g.xlarge` RDS PostgreSQL Primary (~$260/mo)
- 2x `db.r6g.large` RDS PostgreSQL Read Replicas (~$260/mo)
- 3x `cache.r6g.large` ElastiCache Redis cluster (~$360/mo)
- ECS Fargate Workers & SQS, ALB, CloudFront (~$240/mo)

**What if the budget were $500/month?**
If the budget were reduced to $500/month, we would have to aggressively downscale.
- We would not be able to afford the Redis Cluster and the large RDS instances.
- We would be forced to use PostgreSQL for locking (`SELECT FOR UPDATE`) and maybe standard EC2 instances or Lambda instead of ECS.
- The system would be fundamentally unable to serve 5 Lakh concurrent users smoothly at 12:00:00. We'd have to implement aggressive rate-limiting or a virtual waiting room (queueing users *before* they even interact with the system) to throttle the traffic down to the ~500-1000 RPS our smaller database could handle.