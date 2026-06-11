# Architecture Review Q&A

## Question 1: Redis crashes during seat locking

**Question:** If Redis goes down mid-booking, how do you prevent double bookings?

**Answer:** 
- Redis failure → API falls back to PostgreSQL SELECT FOR UPDATE
- Version field in DB catches conflicts (optimistic locking)
- Reduced throughput (3K RPS) but maintains consistency
- Health checks detect Redis failure in 10s

**Gap Identified:** No automatic Redis failover

**Improvement Opportunity:** Add Redis replica with automatic promotion

---

## Question 2: Database bottleneck under peak demand

**Question:** What if 25K RPS overwhelms the database despite caching?

**Answer:**
- Cache hit rate: 92% (only 2K RPS hit DB)
- Partial indexes reduce query time to <50ms
- Connection pool: 500 connections, only 65% used at peak
- Read replicas offload analytics (20% of reads)

**Gap Identified:** No horizontal DB scaling plan

**Improvement Opportunity:** Connection pool monitoring + alert at 80%

---

## Question 3: User attempts to hold 200 seats

**Question:** How do you prevent seat-holding abuse?

**Answer:**
- Currently: No limit (vulnerability)
- Single user could lock entire section
- Blocks legitimate buyers

**Gap Identified:** No per-user seat hold limit

**Improvement Opportunity:** Enforce max 10 seats per booking

---

## Question 4: AWS bill exceeds $2000 budget

**Question:** What causes cost overruns and how to detect?

**Answer:**
- Main risk: Auto-scaling stuck at max (12 instances)
- Data transfer if API responses not compressed
- Lambda workers not terminating

**Gap Identified:** No cost alerting configured

**Improvement Opportunity:** CloudWatch billing alarm at $2100

---

## Question 5: Why Redis locks instead of PostgreSQL row locks?

**Question:** Justify the added complexity of Redis.

**Answer:**
- Postgres locks: 4,400 connections needed at 25K RPS
- max_connections = 500 (exhausted at 3K RPS)
- Redis: 50K ops/sec easily handled
- TTL auto-cleanup prevents lock leaks
- Connection pool preserved for actual DB writes

**Gap Identified:** Redis single point of failure

**Improvement Opportunity:** Redis Cluster with replicas
