# Design Updates

## Update 1: Added Per-User Seat Hold Limit
 
**Triggered by:** Panel Question 3 (seat holding abuse scenario)
 
**What changed:**
- Added Redis counter keyed by user ID (`holds:{userId}:count`) with a TTL of 15 minutes.
- API now checks this counter before attempting to acquire any seat lock.
- If count >= 8, return `429 Too Many Requests` immediately.
- Counter is incremented on successful lock acquisition and decremented when hold is released or booking confirmed.
 
**Why this is necessary:**
Without this, a single user or bot could hold hundreds of seats hostage during peak sale windows, starving legitimate buyers. Our previous IP-based rate limiting was insufficient to stop multi-tab abuse from a single authenticated user.
 
**What it costs:**
One additional Redis GET/INCR per seat hold request (+0.1ms latency), which is negligible compared to the lock acquisition itself (~0.5ms).
 
**What it still doesn't solve:**
A coordinated attack using multiple registered user accounts (Sybils) can still hold seats. True abuse prevention would require verified accounts (e.g., phone OTP or saved payment method requirement before holding), which is a product-level decision.

## Update 2: Added SQS Circuit Breaker Fallback
 
**Triggered by:** Panel Question 4 (SQS outage handling)
 
**What changed:**
- Added a Circuit Breaker pattern to the SQS publish step in the API.
- If SQS publish fails continuously for 60 seconds (or returns 5xx errors), the Circuit Breaker trips to "Open".
- While Open, the API falls back to Synchronous Payment Processing. The HTTP request waits for the payment gateway directly rather than enqueuing.
 
**Why this is necessary:**
If SQS is completely down, our asynchronous queue pipeline breaks, and users are stuck in a "pending" state indefinitely. We would rather accept a severely reduced throughput (due to DB pool exhaustion at ~1,200 RPS) than process zero payments during a major sale. It prioritizes availability over pure performance during a failure scenario.
 
**What it costs:**
Increased API complexity. If the fallback triggers during a 500K user burst, the DB connection pool will exhaust, leading to 503 errors for many users.
 
**What it still doesn't solve:**
If the SQS outage correlates with a DB failure, we still cannot process payments. The fallback only trades a complete outage for a degraded, slow-processing state.
