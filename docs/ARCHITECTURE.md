# ShowTime Architecture Diagram

This diagram represents the complete data flow and system architecture for the ShowTime ticketing backend, designed to handle 500,000 concurrent users.

```text
Mobile / Browser
      │
      ▼
CloudFront CDN ──── [static assets, event pages: cache hit → user]
      │ (API requests only)
      ▼
Application Load Balancer (ALB)
  - SSL termination
  - Health checks every 10s
  - Rate limit: 200 req/IP/min
      │
      ├──── Node.js API ×1
      ├──── Node.js API ×2    ← Auto-scale group (4–20 instances)
      ├──── Node.js API ×3      Scale-out: CPU > 70% for 2 min
      └──── Node.js API ×N
              │
              ├── READ/WRITE ──► Redis Cluster (3 nodes, ElastiCache)
              │                   ├── Cache: availability:event:cat (TTL 30s) ← 30s TTL (see CACHE.md: Seat Availability Counts)
              │                   ├── Cache: event:[id] (TTL 3600s)
              │                   ├── Locks: seat_lock:[id] (TTL 30s, SETNX) ← Redis SETNX lock (see CONCURRENCY.md: 500K RPS demand)
              │                   └── Abuse Prev: holds:[userId]:count (TTL 15m) ← [Update 1] Per-user hold limit
              │
              ├── WRITE ──► PostgreSQL Primary (RDS db.r6g.xlarge)
              │             └── Replicates to ──► Read Replica 1  ← Read replica (see SCHEMA.md: reads separate from write transactions)
              │                                   Read Replica 2
              │
              ├── PUBLISH ──► SQS Payment Queue  ← Async queue (see QUEUE.md: pool exhaustion at 1200 RPS)
              │                Visibility timeout: 30s
              │                Max receive count: 3
              │                DLQ: payment-dlq
              │                     │
              │              ┌──────▼──────────────────┐
              │              │ Payment Worker (ECS) ×10│
              │              │ 1. Read from SQS        │
              │              │ 2. Call Razorpay API    │
              │              │ 3. Update DB            │
              │              │ 4. Publish SNS          │
              │              │ 5. Delete SQS message   │
              │              └─────────────────────────┘
              │                     │
              │              ┌──────▼────────┐
              │              │   AWS SNS     │
              │              ├──► SES Email  │
              │              └──► SNS SMS    │
              │                   (Twilio)   │
              │              └───────────────┘
              │
              └── FALLBACK ──► [Circuit Breaker] Synchronous Payment ← [Update 2] Triggers if SQS fails
                               (Bypasses SQS, API calls Razorpay directly)
```
