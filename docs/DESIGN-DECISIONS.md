# Design Decisions

## Decision 1: Redis SETNX vs PostgreSQL FOR UPDATE

**Context:** Need distributed locking for 25K RPS without exhausting connections.

**Options:**
- A) PostgreSQL SELECT FOR UPDATE
- B) Redis SETNX distributed locks

**Why Chosen:** Redis SETNX
- Postgres exhausts at 3K RPS (needs 4,400 connections at 25K RPS)
- Redis handles 100K+ ops/sec, only 50K needed
- TTL auto-expires locks on crashes

**Tradeoffs:** Separate service to manage, Redis-Postgres consistency gap

**Revision Trigger:** If Redis becomes single point of failure

---

## Decision 2: TTL-based Cache vs Event-driven

**Context:** Cache invalidation strategy for event/seat data.

**Options:**
- A) TTL-only (time-based expiry)
- B) Event-driven invalidation on every write

**Why Chosen:** TTL-based (60s for seat counts, 1h for events)
- Simpler implementation
- Approximate counts acceptable during rush
- Short TTL balances freshness vs load

**Tradeoffs:** Stale data possible for 60s

**Revision Trigger:** If stale data causes user complaints

---

## Decision 3: UUID vs SERIAL for Booking IDs

**Context:** Primary key strategy for distributed system.

**Options:**
- A) SERIAL (auto-increment)
- B) UUID v4

**Why Chosen:** UUID
- Prevents enumeration attacks
- Distributed-safe (no central sequence)
- Merge-friendly for sharding

**Tradeoffs:** 16 bytes vs 8 bytes (larger indexes)

**Revision Trigger:** If index size becomes bottleneck

---

## Decision 4: SQS Visibility Timeout = 30s

**Context:** How long to hide message after worker receives it.

**Options:**
- A) 10s (too short)
- B) 30s
- C) 60s (too long)

**Why Chosen:** 30s
- Payment gateway: 10s timeout
- DB write: 2s
- Buffer: 18s for retries

**Tradeoffs:** Failed workers block messages for 30s

**Revision Trigger:** If payment latency exceeds 20s consistently

---

## Decision 5: Async Payment vs Synchronous

**Context:** Payment gateway takes 3s, would block connections.

**Options:**
- A) Synchronous (wait for payment)
- B) Async via SQS

**Why Chosen:** Async SQS
- Synchronous needs 15K connections (kills app)
- Async reserves seat in 200ms, queues payment
- Retry logic + DLQ for failures

**Tradeoffs:** User waits ~5s for confirmation

**Revision Trigger:** If 5s wait time unacceptable

---

## Decision 6: Read Replicas vs Larger Primary

**Context:** Separate analytics from transactional queries.

**Options:**
- A) Single large primary (db.r5.2xlarge @ $1200/mo)
- B) Primary + 2x read replicas ($580 + $280)

**Why Chosen:** Read replicas
- $360/mo savings
- Analytics don't affect booking writes
- 5s replication lag acceptable for dashboards

**Tradeoffs:** Eventual consistency for reports

**Revision Trigger:** If replication lag > 30s

---

## Decision 7: Redis Seat Holds vs DB-based

**Context:** Where to enforce 10-minute seat hold TTL.

**Options:**
- A) Redis TTL only
- B) DB held_until column only
- C) Both (hybrid)

**Why Chosen:** Hybrid (Redis lock + DB held_until)
- Redis: fast distributed lock (15s TTL)
- DB: authoritative state (10min hold)
- Background job expires DB holds

**Tradeoffs:** Dual management complexity

**Revision Trigger:** If Redis-DB sync issues frequent
