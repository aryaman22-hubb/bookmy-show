# Async Payment Architecture

## Section 1: Why Synchronous Payments Fail

### The Problem
**Synchronous Flow:**
```
Client → API → Seat Lock → Payment Gateway (2-5s) → Confirm Booking → Response
```

**Math:**
- Payment gateway latency: 3s average
- Peak payment RPS: 5,000 (20% of 25,000 RPS)
- Connections held: 5,000 RPS × 3s = **15,000 connections** ❌

**Consequences:**
- Thread pool exhaustion
- Cascading timeouts
- Lost bookings due to client disconnects
- No retry mechanism

### The Solution: Async Queue Pattern
- Reserve seat immediately (200ms)
- Queue payment for background processing
- Client polls for payment status
- Retry logic + DLQ for failures

---

## Section 2: SQS Message Schema

```json
{
  "bookingId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "eventId": "123e4567-e89b-12d3-a456-426614174000",
  "seatIds": [
    "seat-uuid-1",
    "seat-uuid-2"
  ],
  "totalAmount": 1500.00,
  "paymentToken": "tok_1234567890abcdef",
  "idempotencyKey": "booking-550e8400-payment-1",
  "createdAt": "2026-06-10T12:00:15Z",
  "attemptCount": 0
}
```

### Field Explanations

| Field | Purpose |
|-------|---------|
| `bookingId` | Links payment to reservation (foreign key) |
| `userId` | For payment verification + fraud detection |
| `eventId` | Context for logging/analytics |
| `seatIds` | Rollback target if payment fails |
| `totalAmount` | Payment gateway charge amount |
| `paymentToken` | Tokenized card details (PCI-compliant) |
| `idempotencyKey` | Prevents duplicate charges on retries |
| `createdAt` | Timeout calculation (expire after 10 min) |
| `attemptCount` | Circuit breaker after N failures |

**Security Note:** Never send raw card data through SQS. Use tokenized references.

---

## Section 3: Worker Flow

### Success Path
```
1. Worker receives SQS message
2. Validate booking still in 'pending' state
3. Call payment gateway with idempotencyKey
4. Payment succeeds (200 OK)
5. Update DB:
   - bookings.status = 'confirmed'
   - bookings.payment_id = gateway_response.id
   - seats.status = 'booked' (finalize hold)
6. Delete SQS message (ACK)
7. Send confirmation email (SNS topic)
```

### Failure Path
```
1. Worker receives message
2. Payment gateway returns 402 (insufficient funds)
3. Update DB:
   - bookings.status = 'cancelled'
   - seats.status = 'available' (release hold)
4. Delete SQS message (no retry)
5. Send failure notification to user
```

### Retry Path
```
1. Payment gateway times out (504 Gateway Timeout)
2. Do NOT delete SQS message
3. Message becomes visible again after VisibilityTimeout (30s)
4. Worker retries with same idempotencyKey
5. Gateway recognizes duplicate request:
   - If previous succeeded: returns cached response
   - If still pending: continues processing
6. After 3 retries → move to DLQ
```

### DLQ Path (Dead Letter Queue)
```
1. Message fails 3 times (MaxReceiveCount exceeded)
2. SQS moves message to DLQ automatically
3. Manual investigation triggered:
   - Check booking state
   - Verify payment gateway status
   - Refund if double-charged
4. Alert on-call engineer
```

---

## Section 4: Edge Cases

### Edge Case 1: Server Crash After SQS Publish

**Scenario:**
1. API publishes message to SQS
2. Server crashes before responding to client
3. Client retries POST /bookings

**Handling:**
- Use `idempotencyKey` in booking creation
- Database UNIQUE constraint prevents duplicate bookings
- Second request returns existing booking (409 Conflict or 200 with existing data)
- SQS message processes normally

**Pseudocode:**
```sql
INSERT INTO bookings (id, user_id, idempotency_key, ...)
VALUES ($1, $2, $3, ...)
ON CONFLICT (idempotency_key) DO NOTHING
RETURNING id;
```

### Edge Case 2: Payment Timeout (No Response)

**Scenario:**
1. Worker calls payment gateway
2. Request times out after 10s (no response)
3. Unknown if payment succeeded or failed

**Handling:**
- Worker does NOT delete SQS message
- Message retries after VisibilityTimeout
- Use idempotencyKey to query payment status:
  ```javascript
  const status = await gateway.getPaymentStatus(idempotencyKey);
  if (status === 'succeeded') {
    // Complete booking
  } else if (status === 'failed') {
    // Cancel booking
  } else {
    // Still pending, retry later
  }
  ```

**Circuit Breaker:**
- After 3 retries over 5 minutes → assume failure
- Cancel booking, release seats
- Manual reconciliation via DLQ

---

## Section 5: SQS Configuration

```javascript
const queueConfig = {
  QueueName: 'bookmy-payment-queue',
  Attributes: {
    // How long message is invisible after worker receives it
    VisibilityTimeout: '30', // 30 seconds (covers payment call)
    
    // Max times a message is delivered before DLQ
    MaxReceiveCount: '3',
    
    // Dead letter queue ARN
    RedrivePolicy: JSON.stringify({
      deadLetterTargetArn: 'arn:aws:sqs:region:account:bookmy-payment-dlq',
      maxReceiveCount: 3
    }),
    
    // Message retention (14 days max)
    MessageRetentionPeriod: '86400', // 24 hours
    
    // Delay before message becomes available (0 for immediate)
    DelaySeconds: '0'
  }
};
```

### Configuration Rationale

| Parameter | Value | Reasoning |
|-----------|-------|-----------|
| VisibilityTimeout | 30s | Payment gateway call (10s) + DB write (2s) + buffer |
| MaxReceiveCount | 3 | Balance between retry attempts and fast failure |
| MessageRetentionPeriod | 24h | Bookings expire after 10 min; long retention for debugging |
| DLQ | Enabled | Critical for payment reconciliation audits |

### Worker Scaling
- **Initial:** 5 Lambda workers (reserve concurrency)
- **Peak:** Auto-scale to 50 workers (5,000 payments / 60s / 2s per payment = 167 workers needed, but spread over 10 min = 50 concurrent)
- **Cost:** ~$100/month for 5K payments @ 2s duration

### Monitoring Alerts
- DLQ depth > 10 messages → PagerDuty alert
- Queue age > 5 minutes → Investigate worker health
- Payment success rate < 95% → Check gateway status
