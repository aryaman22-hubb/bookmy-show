# Redis Cache Architecture

## Cache Strategy: Cache-Aside Pattern

### Cache Entries

#### 1. Event Details
```
Key: event:{eventId}
TTL: 3600s (1 hour)
Invalidation: Event update, deletion
Pattern: Read-through

Value:
{
  "id": "uuid",
  "title": "Movie Name",
  "venue": "Theater X",
  "eventTime": "ISO8601",
  "bookingOpensAt": "ISO8601"
}
```

**Why Cache?**
- Read-heavy (1000:1 read-write ratio)
- Immutable after creation (rarely updated)
- Reduces DB load by 95%

---

#### 2. Seat Availability Count
```
Key: seat_count:{eventId}
TTL: 60s (1 minute)
Invalidation: Any booking, cancellation
Pattern: Write-through with short TTL

Value: "423" (available seat count)
```

**Why Cache?**
- Displayed on event listing page (high traffic)
- Exact count not critical (approximate is acceptable)
- Short TTL prevents stale data during rush

**Update Flow:**
```
On booking:
  1. Update DB (seats.status = 'booked')
  2. DECR seat_count:{eventId}
  3. If key missing, skip (will refresh on next read)
```

---

#### 3. Static Seat Layout
```
Key: seat_layout:{eventId}
TTL: 86400s (24 hours)
Invalidation: Manual (event creation only)
Pattern: Lazy load

Value:
{
  "sections": [
    {
      "name": "VIP",
      "rows": ["A", "B"],
      "seatsPerRow": 10,
      "basePrice": 500
    }
  ]
}
```

**Why Cache?**
- Seat grid structure never changes per event
- Required for UI rendering
- Heavy JSON payload (10KB+)

---

### ❌ Individual Seat Status NOT Cached

**Why?**
- **Consistency nightmare:** Redis and Postgres diverge under race conditions
- **Cache invalidation cost:** Updating 500 cache keys per booking kills throughput
- **TTL dilemma:** Short TTL defeats caching; long TTL shows stale data
- **Solution:** Query DB with `idx_seats_event_status` index (fast enough)

**Trade-off:**
- Direct DB queries for seat status checks
- Index covers query (no table scan)
- Acceptable latency: <50ms for 100 seats

---

## Cache-Aside Invalidation Flow

### Pseudocode

```javascript
// READ PATH
async function getEventDetails(eventId) {
  const cacheKey = `event:${eventId}`;
  
  // Try cache first
  let event = await redis.get(cacheKey);
  if (event) {
    return JSON.parse(event);
  }
  
  // Cache miss: fetch from DB
  event = await db.query('SELECT * FROM events WHERE id = $1', [eventId]);
  
  // Populate cache
  await redis.setex(cacheKey, 3600, JSON.stringify(event));
  
  return event;
}

// WRITE PATH (invalidate on update)
async function updateEvent(eventId, updates) {
  await db.query('UPDATE events SET ... WHERE id = $1', [eventId]);
  
  // Invalidate cache
  await redis.del(`event:${eventId}`);
  
  // Next read will refresh from DB
}

// SEAT COUNT UPDATE
async function bookSeats(eventId, seatIds) {
  // Update DB in transaction
  await db.query('UPDATE seats SET status = $1 WHERE id = ANY($2)', ['booked', seatIds]);
  
  // Update cache counter
  await redis.decrby(`seat_count:${eventId}`, seatIds.length);
  
  // If cache missing (TTL expired), ignore error
}

// BACKGROUND JOB: Refresh seat counts
async function refreshSeatCounts() {
  const events = await db.query(`
    SELECT event_id, COUNT(*) as available
    FROM seats
    WHERE status = 'available'
    GROUP BY event_id
  `);
  
  for (const event of events) {
    await redis.setex(
      `seat_count:${event.event_id}`,
      60,
      event.available
    );
  }
}
```

---

## Cache Configuration

### Redis Cluster Setup
- **Instance:** cache.r6g.large (13.5 GB RAM)
- **Eviction policy:** `allkeys-lru` (least recently used)
- **Persistence:** Disabled (cache is disposable)
- **Connection pool:** 50 connections per app server

### Monitoring Metrics
- Hit rate target: >90%
- P99 latency: <5ms
- Memory usage: <70% (avoid eviction thrashing)

### Failure Handling
```javascript
async function getCachedData(key, fallback) {
  try {
    return await redis.get(key);
  } catch (err) {
    logger.warn('Redis unavailable, fallback to DB');
    return await fallback(); // Bypass cache
  }
}
```

**Philosophy:** Cache failures degrade performance, not availability.
