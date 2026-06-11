# System Architecture

## Production Architecture Diagram with Labelled Flows

```
┌─────────────────────────────────────────────────────────────────┐
│                          INTERNET                               │
│                    (500K Concurrent Users)                      │
└────────────────────────────┬────────────────────────────────────┘
                             │
                             │ HTTPS Requests
                             │ Peak: 25,000 RPS (see README.md)
                             ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CloudFront CDN                               │
│             (Global Edge Cache - $180/month)                    │
│  • Static assets (HTML/CSS/JS/Images)                          │
│  • Event listing page cache (TTL: 60s)                         │
│  • Seat layout JSON cache (TTL: 24h)                           │
└─────────────┬─────────────────────────┬─────────────────────────┘
              │                         │
        cache hit                  cache miss
         (50ms)                     (continues)
              │                         │
              ▼                         ▼
         Response              ┌────────────────────┐
                              │ Application Load   │
                              │    Balancer        │
                              │ ($180/month incl)  │
                              └─────────┬──────────┘
                                        │
                    HTTPS + Health Checks (every 10s)
                                        │
         ┌──────────────────────────────┼────────────────────────┐
         │                              │                        │
         ▼                              ▼                        ▼
┌─────────────────┐         ┌────────────────────┐    ┌────────────────┐
│  Node.js API    │         │   Node.js API      │    │  Node.js API   │
│  t3.medium (1)  │   ...   │   t3.medium (2)    │    │ t3.medium (6)  │
│  $250/mo total  │         │                    │    │  (Auto-Scale)  │
└────────┬────────┘         └──────────┬─────────┘    └────────┬───────┘
         │                             │                        │
         └─────────────────────────────┼────────────────────────┘
                                       │
         ┌─────────────────────────────┼─────────────────────────┐
         │                             │                         │
         │ Redis Ops:                  │ DB Queries:             │
         │ • SETNX lock:seat:{id}      │ • UPDATE seats          │
         │ • GET event:{id}            │ • INSERT bookings       │
         │ • DECR seat_count:{id}      │ • SELECT FOR reads      │
         │                             │                         │
         ▼                             ▼                         │
┌──────────────────────┐    ┌────────────────────────┐          │
│  ElastiCache Redis   │    │  RDS PostgreSQL        │          │
│  cache.r6g.large     │    │  db.r5.xlarge          │          │
│  ($260/month)        │    │  ($580/month)          │          │
│                      │    │                        │          │
│ ← Redis SETNX lock   │    │ ← Authoritative state  │          │
│   (CONCURRENCY.md)   │    │   (SCHEMA.md)          │          │
│                      │    │                        │          │
│ ← Event cache        │    │ ← UUID primary keys    │          │
│   (CACHE.md)         │    │   (SCHEMA.md)          │          │
│                      │    │                        │          │
│ ← Seat count cache   │    │ ← held_until TTL       │          │
│   TTL: 60s           │    │   (SCHEMA.md)          │          │
│   (CACHE.md)         │    │                        │          │
└──────────────────────┘    └───────────┬────────────┘          │
                                        │                        │
                               Async Replication                 │
                                  (lag: ~5s)                     │
                                        │                        │
                    ┌───────────────────┴───────────────┐        │
                    │                                   │        │
                    ▼                                   ▼        │
          ┌──────────────────┐              ┌──────────────────┐ │
          │  Read Replica 1  │              │  Read Replica 2  │ │
          │  db.t4g.large    │              │  db.t4g.large    │ │
          │  ($140/mo each)  │              │                  │ │
          │                  │              │                  │ │
          │ ← Analytics only │              │ ← Dashboards     │ │
          │   (SCHEMA.md)    │              │   (README.md)    │ │
          └──────────────────┘              └──────────────────┘ │
                                                                  │
         ┌────────────────────────────────────────────────────────┘
         │
         │ Publish Payment Message
         │ (Non-blocking, <5ms)
         │
         ▼
┌──────────────────────────────────────────────────────────────────┐
│                   ASYNC PAYMENT PIPELINE                         │
│                                                                  │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │ SQS Queue: bookmy-payment-queue                          │  │
│   │ ($150/month incl workers)                                │  │
│   │                                                          │  │
│   │ ← Async payments (QUEUE.md)                             │  │
│   │ ← Visibility timeout: 30s (QUEUE.md Section 5)          │  │
│   │ ← MaxReceiveCount: 3                                    │  │
│   └────────────────┬──────────────────────┬──────────────────┘  │
│                    │                      │                     │
│         Message available        Message failed 3x              │
│           (poll every 1s)          (MaxReceiveCount)            │
│                    │                      │                     │
│                    ▼                      ▼                     │
│         ┌─────────────────────┐   ┌──────────────────┐         │
│         │  Lambda Workers     │   │   SQS DLQ        │         │
│         │  (Pool: 5-50)       │   │   (Manual Fix)   │         │
│         │  Auto-scale         │   │                  │         │
│         └──────────┬──────────┘   │ ← Failed         │         │
│                    │               │   payments       │         │
│      Process payment               │   (QUEUE.md)     │         │
│      with idempotency              └──────────────────┘         │
│                    │                                            │
│                    ▼                                            │
│         ┌─────────────────────┐                                │
│         │ Payment Gateway API │                                │
│         │ (Stripe/Razorpay)   │                                │
│         │ External - 3s P99   │                                │
│         │                     │                                │
│         │ ← Idempotency key   │                                │
│         │   (QUEUE.md)        │                                │
│         └──────────┬──────────┘                                │
│                    │                                            │
│         ┌──────────┴──────────┐                                │
│         │                     │                                │
│    Success (200)         Fail (402)                            │
│         │                     │                                │
│         ▼                     ▼                                │
│   Update booking       Cancel booking                          │
│   status='confirmed'   status='cancelled'                      │
│   Release Redis lock   Release seats                           │
│         │                     │                                │
│         ▼                     ▼                                │
│   ┌──────────────────────────────────────┐                    │
│   │  SNS Topic: booking-notifications    │                    │
│   │                                       │                    │
│   │  ← Success/failure alerts            │                    │
│   └─────────┬────────────────────────────┘                    │
│             │                                                  │
│   ┌─────────┴────────────┐                                    │
│   │                      │                                    │
│   ▼                      ▼                                    │
│  SES Email          SMS Gateway                               │
│  Confirmation       (Twilio/SNS)                              │
│                                                               │
└───────────────────────────────────────────────────────────────┘


MONITORING & OBSERVABILITY
┌────────────────────────────────────────────────────┐
│  CloudWatch Metrics + X-Ray Distributed Tracing    │
│  • RPS, Latency (P50/P99)                         │
│  • Redis hit rate                                  │
│  • DB connection pool usage                        │
│  • SQS queue depth + DLQ depth (ALARM >1000)     │
│  • Payment success rate + circuit breaker status   │
│  • Cost alarm ($2100 threshold)                    │
│  ($100/month - included in S3 + Misc)             │
└────────────────────────────────────────────────────┘

POST-REVIEW IMPROVEMENTS (Part B):
┌────────────────────────────────────────────────────┐
│  1. Seat Hold Limiter: max 10 seats/user          │
│  2. Redis Replica: Multi-AZ failover (+$130/mo)   │
│  3. Queue Depth Alarms: SQS >1000, DLQ >10        │
│  4. Payment Circuit Breaker: fail after 3 timeouts│
└────────────────────────────────────────────────────┘
```

---

## Component Reference Table

| Component | Purpose | Scaling Strategy | Related Part A Decision |
|-----------|---------|------------------|------------------------|
| **CloudFront CDN** | Cache static assets + event listings at edge locations | AWS-managed global auto-scale | Cache-aside pattern (CACHE.md) |
| **Application Load Balancer** | Distribute traffic across Node.js instances | AWS-managed horizontal scaling | Stateless API design (README.md) |
| **Node.js API Servers** | Handle booking/event/user API requests | Horizontal auto-scale (2-12 instances) | 25K RPS target (README.md) |
| **ElastiCache Redis** | Distributed locks + cache layer | Vertical scaling (instance upgrade) | SETNX locking (CONCURRENCY.md) |
| **PostgreSQL Primary** | Authoritative seat inventory + bookings | Vertical scaling (32GB RAM, 4 vCPU) | UUID keys + version (SCHEMA.md) |
| **Read Replicas (2x)** | Offload analytics/dashboard queries | Async replication from primary | Analytics separation (SCHEMA.md) |
| **SQS Payment Queue** | Decouple booking from payment processing | AWS-managed infinite throughput | Async payments (QUEUE.md) |
| **Lambda Workers** | Process payments with retry logic | Auto-scale 5-50 concurrent executions | IdempotencyKey (QUEUE.md) |
| **SQS DLQ** | Manual reconciliation for failed payments | AWS-managed retention (24h) | MaxReceiveCount=3 (QUEUE.md) |
| **SNS + SES** | Send booking confirmations via email/SMS | AWS-managed pub/sub | Post-payment notifications |
| **CloudWatch + X-Ray** | Monitor metrics + distributed tracing | AWS-managed observability | Operational visibility |

---

## Component Table

| Component | Purpose | Scaling Strategy |
|-----------|---------|------------------|
| **CloudFront** | CDN for static assets + API response caching | Global edge locations (auto-scaled) |
| **ALB** | Load distribution + SSL termination | AWS-managed, auto-scales |
| **EC2 Auto Scaling** | API servers for booking logic | Horizontal: 2→12 instances based on CPU/RPS |
| **Node.js Services** | RESTful API (Express.js) | Stateless, scale out during peaks |
| **Redis Cluster** | Distributed locks + cache layer | Vertical scaling (upgrade instance size) |
| **PostgreSQL Primary** | Authoritative seat inventory | Vertical scaling (32GB RAM) |
| **Read Replicas (2x)** | Analytics, reporting, user dashboards | Separate read traffic from writes |
| **SQS** | Payment message queue | AWS-managed, infinite throughput |
| **Lambda Workers** | Async payment processing | Auto-scale 0→50 concurrent executions |
| **S3** | Backup storage, logs | AWS-managed, infinite storage |

---

## Traffic Flow

### Read-Heavy Path (Event Browsing)
```
User → CloudFront (cache hit) → Response (50ms)
User → CloudFront (miss) → ALB → Node.js → Redis (cache hit) → Response (100ms)
User → All miss → ALB → Node.js → PostgreSQL Read Replica → Response (200ms)
```

### Write Path (Seat Booking)
```
User → ALB → Node.js → Redis SETNX lock → PostgreSQL Write → SQS publish → Response (300ms)
Background: SQS → Lambda → Payment Gateway → PostgreSQL Update
```

### Data Consistency Flow
```
Write to PostgreSQL → Invalidate Redis cache → Next read refreshes from DB
```

---

## High Availability

- **Multi-AZ Deployment:** RDS + Redis across 3 availability zones
- **Auto-failover:** RDS promotes read replica to primary (60s RTO)
- **Health Checks:** ALB removes unhealthy instances in 30s
- **Circuit Breaker:** Skip Redis on failure, direct DB queries

---

## Monitoring Stack

- **CloudWatch:** CPU, memory, disk I/O, query latency
- **X-Ray:** Distributed tracing for API requests
- **RDS Performance Insights:** Slow query detection
- **Custom Metrics:** Booking success rate, lock contention, queue depth
