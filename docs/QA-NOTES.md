# Q&A Notes from Panel Roast

**1. "What happens if Redis crashes mid-lock?"**
- **My Answer:** We have three layers of protection. First, Redis Cluster has 3 nodes for HA. Second, if the cluster is down, the API fails to acquire the lock and returns a 503 rather than proceeding. Third, even if it falls through, PostgreSQL's `version` column acts as an optimistic lock. The second concurrent update gets a version mismatch error. Double-bookings are prevented, though throughput degrades.
- **Self-Assessment:** Good answer. Addressed the failure mode accurately using components designed in Part A.

**2. "Your read replicas have up to 500ms replication lag. A user sees 'available' and tries to book - what happens?"**
- **My Answer:** Replicas are only used for reads (like availability counts). When a user actually clicks "Book", the request routes to the primary DB where the exact seat status is checked and the `version` is verified under a Redis lock. If the seat is booked, they get an error. Showing stale data on the UI is a UX inconvenience, not a system failure.
- **Self-Assessment:** Solid answer. Properly separated read paths from transactional write paths.

**3. "What stops one user from holding 200 seats?"**
- **My Answer:** Currently, we don't have a specific mechanism to prevent a single user from calling the hold API repeatedly, but they would be limited by IP rate limiting on the ALB (200 req/min).
- **Self-Assessment:** Weak answer. IP rate limiting doesn't stop 20 browser tabs, and "we don't have a mechanism" is exactly the kind of vulnerability the panel looks for. Need to fix this in the post-roast updates by adding a Redis counter per user.

**4. "SQS is down. Your payment queue is unavailable. The API is returning 202 'Booking in progress' but nothing is actually being processed. How does the user know? How do you recover?"**
- **My Answer:** We can add a CloudWatch alarm to detect when SQS publish fails or queue depth drops to 0. Recovery is handled automatically once SQS comes back, as workers will resume. If it's totally down, we'd need a fallback.
- **Self-Assessment:** Missing a fallback strategy. Need to implement a circuit breaker to switch to synchronous payments at reduced throughput if SQS is completely unavailable.

**5. "Your auto-scaling spikes your bill to $3,200 this month - what's your plan?"**
- **My Answer:** The $2,000 budget is for steady-state. Event spikes are budgeted as a cost of goods sold against ticket revenue. However, to optimize, we can use Spot Instances for the ECS workers and auto-scale API nodes, since they are stateless, saving up to 70%.
- **Self-Assessment:** Reasonable answer. Framing infrastructure cost as a tiny percentage of event revenue (e.g., Coldplay ticket sales) works well.
