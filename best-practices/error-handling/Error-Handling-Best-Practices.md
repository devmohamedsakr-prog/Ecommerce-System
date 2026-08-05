# Error Handling Best Practices

**Status:** Best Practice Guide | **Priority:** CRITICAL | **Scope:** Application & Infrastructure

---

## Overview

Error handling determines system reliability. Poor error handling: cascading failures, data loss, unhappy customers. Good error handling: graceful degradation, quick recovery, detailed diagnostics. Separates 99.9% uptime from 99.99% uptime.

## Business Impact

- Unhandled errors cause cascading failures (one service down → entire system down)
- Proper error handling keeps system running with reduced capacity
- Example: Payment service fails → handle gracefully, users notified, orders queued for retry
- Cost: Proper error handling prevents $9,000/second downtime

## Error Categories

### 1. Transient Errors (Retry-able)
```
Network timeout → Connection failed temporarily
Service temporarily overloaded → 503 error
Database connection pool exhausted → Try again in 100ms

Recovery: Automatic retry with backoff
Max retries: 3-5 times
Backoff: 100ms, 200ms, 400ms (exponential)
```

### 2. Permanent Errors (Non-Retry-able)
```
Invalid input → 400 Bad Request
Resource not found → 404 Not Found
Authentication failed → 401 Unauthorized

Recovery: User action required or logging only
Action: Return clear error message to user
```

### 3. Catastrophic Errors (Alert)
```
Database down → System degraded
Payment processor down → Orders can't complete
Infrastructure failure → Multiple services affected

Recovery: Alert ops team, graceful degradation
Action: Activate incident response plan
```

## Pattern 1: Circuit Breaker

```
Purpose: Prevent cascading failures
How: Track success/failure rate
Action: Open circuit on high failure rate

States:
1. CLOSED (normal)
   - Requests flow normally
   - Success rate tracked
   
2. OPEN (circuit broken)
   - Consecutive failures: 5
   - Circuit opens: Stop sending requests
   - Fail fast: Return error immediately
   - Benefit: Prevents overloading failing service
   
3. HALF-OPEN (recovery test)
   - After 30 seconds: Try one request
   - If succeeds: Close circuit, resume traffic
   - If fails: Back to OPEN, wait another 30s

Example: Payment service fails
- First fail: Attempt 2
- Fifth fail: Open circuit
- All subsequent calls fail immediately (no wait)
- Users see: "Payment temporarily unavailable"
- No queue buildup on payment service
- Service can recover
```

### Circuit Breaker Implementation

```javascript
class CircuitBreaker {
  constructor(operation, threshold = 5, timeout = 30000) {
    this.operation = operation;
    this.failureCount = 0;
    this.threshold = threshold;
    this.timeout = timeout;
    this.state = 'CLOSED';
    this.nextAttempt = Date.now();
  }

  async execute(args) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker OPEN');
      }
      this.state = 'HALF_OPEN';
    }

    try {
      const result = await this.operation(...args);
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;
    this.state = 'CLOSED';
  }

  onFailure() {
    this.failureCount++;
    if (this.failureCount >= this.threshold) {
      this.state = 'OPEN';
      this.nextAttempt = Date.now() + this.timeout;
    }
  }
}

// Usage
const paymentBreaker = new CircuitBreaker(
  async (orderId) => await payment.process(orderId),
  5,  // fail after 5 errors
  30000  // wait 30s before retry
);

try {
  const result = await paymentBreaker.execute([orderId]);
} catch (error) {
  // Handle: payment temporarily unavailable
  // Queue for manual processing later
}
```

## Pattern 2: Retry with Exponential Backoff

```
Purpose: Handle transient failures
Strategy: Retry with increasing delay

Attempt 1: Immediate
Attempt 2: Wait 100ms + random jitter (0-100ms)
Attempt 3: Wait 200ms + random jitter (0-200ms)
Attempt 4: Wait 400ms + random jitter (0-400ms)
Attempt 5: Give up

Jitter: Prevents thundering herd (all clients retry simultaneously)

Which errors to retry:
✓ RETRY: 408 (timeout), 429 (rate limit), 503 (service unavailable), 504 (gateway timeout)
✗ DON'T RETRY: 400 (bad request), 401 (unauthorized), 404 (not found)

Example: Timeout retrieving product
- Attempt 1: Fails (timeout)
- Wait: 100-200ms
- Attempt 2: Succeeds, return product
```

### Retry Implementation

```javascript
async function retryWithBackoff(operation, maxAttempts = 5) {
  let lastError;
  
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await operation();
    } catch (error) {
      lastError = error;
      
      // Don't retry permanent errors
      if ([400, 401, 403, 404].includes(error.statusCode)) {
        throw error;
      }
      
      if (attempt < maxAttempts) {
        // Exponential backoff + jitter
        const delay = Math.min(100 * Math.pow(2, attempt - 1), 10000);
        const jitter = Math.random() * delay;
        await sleep(delay + jitter);
      }
    }
  }
  
  throw lastError;
}

// Usage
const product = await retryWithBackoff(
  () => productService.getProduct(productId),
  5
);
```

## Pattern 3: Timeouts

```
Purpose: Prevent hanging requests
Strategy: Set maximum wait time

Connection timeout: 5 seconds (can't reach service)
Request timeout: 30 seconds (service responding but slow)
Shutdown timeout: 10 seconds (give requests time to finish)

Without timeout:
- Slow service takes 2 minutes to respond
- Client waits 2 minutes
- If 1000 concurrent requests: 2000 requests queued
- System resource exhaustion

With timeout:
- Request takes > 30s: Fail and retry
- Fail fast: Free up resources
- Total time: 5-10 seconds vs 2 minutes
```

### Timeout Implementation

```javascript
async function withTimeout(promise, ms) {
  let timeoutId;
  
  const timeoutPromise = new Promise((_, reject) =>
    timeoutId = setTimeout(
      () => reject(new Error('Timeout')),
      ms
    )
  );
  
  try {
    return await Promise.race([promise, timeoutPromise]);
  } finally {
    clearTimeout(timeoutId);
  }
}

// Usage
const result = await withTimeout(
  fetchProduct(productId),
  30000  // 30 second timeout
);
```

## Pattern 4: Graceful Degradation

```
Purpose: Keep system running when components fail

Scenario: Product service fails
- Normal: Show products to users
- Degraded: Show cached products (1 day old)
- Severely degraded: Show generic "Products unavailable"
- Never: Complete outage

Implementation:
1. Try primary data source
2. If fails: Try cache
3. If cache empty: Show degraded view
4. Alert ops team

Example: Recommendation engine fails
- Primary: Real-time recommendations from ML service
- Fallback: Pre-computed trending products
- Further fallback: Random products
- Cost: Slightly worse UX, but system stays up
```

### Graceful Degradation Example

```javascript
async function getRecommendations(userId) {
  try {
    // Try primary: Real-time ML recommendations
    return await mlService.getRecommendations(userId);
  } catch (error) {
    logger.error('ML service failed', error);
    
    try {
      // Fallback: Pre-computed trending
      return await cache.get(`trending:${userId.region}`);
    } catch (cacheError) {
      logger.error('Cache lookup failed', cacheError);
      
      // Final fallback: Generic recommendations
      return getGenericRecommendations();
    }
  }
}
```

## Pattern 5: Error Tracking & Alerting

```
Error tracking levels:

Level 1: LOG
- Expected errors: Invalid input, not found
- Action: Log for analytics

Level 2: WARN
- Transient errors after retries: Timeout after 5 retries
- Action: Log, alert on spike

Level 3: ERROR
- Persistent failures: Service down for 5+ minutes
- Action: Page on-call engineer

Level 4: CRITICAL
- Data loss, security breach, complete outage
- Action: Page on-call + manager

Implementation:
1. Categorize every error
2. Route to appropriate alerting level
3. Include context: service, user, timestamp, stack trace
4. Alert rules: If 50 ERRORs in 5 min → page engineer
```

### Error Tracking Implementation

```javascript
const errorLevels = {
  LOG: { logLevel: 'info', alert: false },
  WARN: { logLevel: 'warn', alert: false, alertThreshold: 10 },
  ERROR: { logLevel: 'error', alert: true, alertThreshold: 5 },
  CRITICAL: { logLevel: 'error', alert: true, alertThreshold: 1 }
};

function handleError(error, context) {
  const level = categorizeError(error);
  const config = errorLevels[level];
  
  logger.log(config.logLevel, {
    message: error.message,
    stack: error.stack,
    context,
    level
  });
  
  if (config.alert) {
    // Send to alerting system
    alerting.sendAlert({
      level,
      message: error.message,
      context,
      threshold: config.alertThreshold
    });
  }
}

function categorizeError(error) {
  if (error.statusCode === 404) return 'LOG';
  if (error.statusCode === 429) return 'WARN';
  if (error.message.includes('timeout')) return 'ERROR';
  if (error.message.includes('data loss')) return 'CRITICAL';
  return 'ERROR';
}
```

## Pattern 6: Error Recovery Procedures

```
1. Detect error
2. Log error with context
3. Attempt recovery:
   - Retry (if transient)
   - Circuit breaker (if service failing)
   - Fallback (if cache available)
   - Graceful degradation (if possible)
4. If recovery succeeds: Continue normally
5. If recovery fails: Alert ops
6. Manual recovery:
   - Ops investigates
   - Fixes root cause
   - Restarts service
   - Manual replay of failed requests
```

## Monitoring & Metrics

```
Track:
- Error rate by service (target: < 0.1%)
- Error rate by type (4xx vs 5xx)
- Recovery success rate (target: > 95%)
- Time to recovery (target: < 5 minutes)
- Circuit breaker opens (should be rare)
- Retry rate (target: < 2%)

Alerts:
- Error rate > 1%
- Circuit breaker open for > 5 min
- Service unavailable for > 5 min
- Time to recovery > 15 min
```

## Error Response Format

```json
{
  "error": {
    "code": "PAYMENT_FAILED",
    "message": "Payment processing failed. Please try again.",
    "statusCode": 503,
    "retryable": true,
    "retryAfter": 5,
    "details": {
      "reason": "Payment processor timeout",
      "requestId": "req_12345"
    }
  }
}
```

---

**Guide Version:** 1.0 | **Status:** Production Best Practice | **Impact:** 99.9% → 99.99% uptime possible

