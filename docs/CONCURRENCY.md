# Concurrency Control Analysis

## Option A: PostgreSQL SELECT FOR UPDATE

### Implementation
```sql
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Lock specific seats (row-level lock)
SELECT id, status, version 
FROM seats 
WHERE id = ANY($1::uuid[]) 
  AND event_id = $2 
  AND status = 'available'
FOR UPDATE NOWAIT;

-- If all seats locked and available:
UPDATE seats 
SET status = 'held', 
    held_until = NOW() + INTERVAL '10 minutes',
    version = version + 1
WHERE id = ANY($1::uuid[]);

INSERT INTO bookings (...) VALUES (...);
INSERT INTO booking_seats (...) VALUES (...);

COMMIT;
```

### Deadlock Analysis
- **Risk:** User A locks seat 1→2, User B locks seat 2→1 (deadlock)
- **Mitigation:** 
  - `NOWAIT` fails immediately instead of waiting
  - Client retries with exponential backoff
  - Lock seats in sorted order by ID

### Capacity Calculation

**Formula:**
```
Connections Held = (Regular RPS × Query Time) + (Payment RPS × Payment Hold Time)
```

**Given:**
- max_connections = 500
- 80% regular requests @ 20ms query time
- 20% payment requests @ 800ms hold time

**At 25,000 RPS:**
- Regular: 25,000 × 0.8 × 0.02s = 400 connections
- Payment: 25,000 × 0.2 × 0.8s = 4,000 connections
- **Total: 4,400 connections REQUIRED** ❌

**Connection Pool Exhaustion at ~3,000 RPS**

### Verdict
✅ Excellent consistency guarantees  
❌ **FAILS at scale** - connection pool exhaustion

---

## Option B: Redis SETNX Distributed Lock

### Implementation
```javascript
// Lock acquisition
const lockKey = `lock:seat:${seatId}`;
const lockValue = `${bookingId}:${timestamp}`;
const acquired = await redis.set(lockKey, lockValue, 'NX', 'EX', 15);

if (!acquired) {
  throw new Error('Seat already locked');
}

// Critical section: update PostgreSQL
try {
  await db.query('UPDATE seats SET status = $1 WHERE id = $2', ['held', seatId]);
  await db.query('INSERT INTO bookings ...');
} finally {
  // Lua script for safe unlock (only owner can delete)
  await redis.eval(`
    if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
    else
      return 0
    end
  `, 1, lockKey, lockValue);
}
```

### TTL Strategy
- Lock TTL: 15 seconds (covers DB write + network latency)
- Seat hold TTL: 10 minutes (user decision time)
- If process crashes, Redis auto-expires lock

### Failure Scenarios

**Scenario 1: Worker crashes after lock, before DB write**
- Redis lock expires after 15s
- Seat remains `available` (no DB update committed)
- ✅ Safe: seat auto-released

**Scenario 2: Network partition during unlock**
- Lock expires naturally via TTL
- ✅ Safe: lock doesn't leak

**Scenario 3: Clock skew extends TTL**
- Use Redis server time, not client time
- ✅ Mitigated

### Capacity Calculation
- Redis can handle 100,000+ ops/sec on single node
- Lock operations: 2 per booking (acquire + release)
- At 25,000 RPS: **50,000 Redis ops/sec** ✅

---

## Option C: Hybrid Approach (CHOSEN)

### Architecture
1. **Redis for seat locking** (high throughput)
2. **PostgreSQL for state persistence** (ACID guarantees)
3. **Version field for optimistic locking fallback**

### Booking Flow
```
1. Client → POST /bookings/reserve
2. API → Redis SETNX lock for each seat
3. API → PostgreSQL UPDATE (if lock acquired)
4. API → Check version conflict (optimistic lock)
5. API → SQS message for async payment
6. Return booking_id + payment_token
```

### Why Hybrid Wins

| Aspect | Postgres Only | Redis Only | Hybrid |
|--------|--------------|------------|--------|
| Throughput | 3,000 RPS | 100,000 RPS | 25,000 RPS |
| Consistency | Strong | Weak | Strong |
| Connection pool | Exhausted | N/A | Preserved |
| Failure recovery | Perfect | Eventual | Strong + Fast |
| Cost | High (large instance) | Medium | **Optimized** |

### Final Justification
- Redis handles lock contention (distributed mutex)
- PostgreSQL persists authoritative state
- Version field detects Redis-Postgres desync
- Connection pool used only for writes, not locks
- Scales to 25,000 RPS within budget constraints
