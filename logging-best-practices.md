# Logging Best Practices: Beyond the How

## Overview

Logging is not just about recording what happened—it's a strategic tool for understanding system behavior, debugging issues, and making data-driven decisions about your application's health and performance. This guide covers **why** to log, **when** to log, **what** to log, and **how** to extract value from logs.

---

## Why Log? The Strategic Value

### 1. **Debugging & Troubleshooting**
- **The Problem:** Production bugs appear without warning, often in scenarios you can't reproduce locally
- **The Value:** Logs are your time machine—they show exactly what happened before the failure
- **Example:** A payment processing failure. Without logs, you're guessing. With logs, you see the exact API call, response, and state at that moment

### 2. **Monitoring & Alerting**
- **The Problem:** You can't watch your system 24/7
- **The Value:** Logs enable automated monitoring to detect anomalies (spikes, errors, slow responses)
- **Example:** Monitor logs for failed authentication attempts to trigger alerts on potential security issues

### 3. **Compliance & Audit Trails**
- **The Problem:** Regulatory requirements demand proof of who did what and when
- **The Value:** Structured logs create an audit trail for sensitive operations
- **Example:** GDPR/HIPAA compliance requires logging access to user data with timestamps and user IDs

### 4. **Performance Optimization**
- **The Problem:** You don't know which parts of your system are slow
- **The Value:** Logs with timing data reveal bottlenecks and optimization opportunities
- **Example:** Log database query times to identify slow queries worth optimizing

### 5. **Understanding User Behavior**
- **The Problem:** Analytics tools are expensive; you're missing context
- **The Value:** Logs reveal the sequence of actions users take, helping identify friction points
- **Example:** Track page transitions and form field interactions to understand where users drop off

### 6. **Proactive Issue Detection**
- **The Problem:** Waiting for users to report bugs is reactive and costly
- **The Value:** Pattern detection in logs can surface issues before they impact users
- **Example:** Detect increasing error rates or memory leaks before they cause outages

---

## When to Log? Identifying Key Moments

### Tier 1: Critical Operations (Always Log)
These are high-impact events that directly affect users or system stability.

```
- Application startup/shutdown
- Authentication attempts (success/failure)
- Authorization decisions (especially denials)
- Database transactions (start, commit, rollback)
- External API calls (request/response)
- Payment or financial transactions
- Data deletion or modification of sensitive data
- System configuration changes
- Service crashes or exceptions
```

**Why?** If something goes wrong with these, you need complete visibility.

### Tier 2: Important Business Logic (Log Strategically)
These are normal operations but worth tracking for insights.

```
- Significant state transitions (e.g., order status changes)
- Cache hits/misses
- User actions that trigger calculations
- Workflow completions
- Email or notification sends
- File uploads/downloads
- Background job executions
```

**Why?** These logs help you understand system behavior and performance patterns.

### Tier 3: Development/Testing (Log in Non-Production)
These are noisy but useful during development.

```
- Variable assignments and function arguments
- Loop iterations
- Conditional branches taken
- Temporary debugging statements
```

**Why?** Too much detail in production noise is dangerous; remove or make configurable.

### When NOT to Log

```
- ❌ Don't log every HTTP request (use access logs instead)
- ❌ Don't log sensitive data (passwords, API keys, tokens, SSNs)
- ❌ Don't log in tight loops without sampling
- ❌ Don't log at the wrong level (ERROR when it should be WARNING)
```

---

## What to Log? Structured Information

### The Anatomy of a Good Log Entry

```json
{
  "timestamp": "2024-02-28T15:30:45.123Z",
  "level": "ERROR",
  "service": "payment-service",
  "traceId": "abc123def456",
  "userId": "user_789",
  "action": "charge_card",
  "status": "failed",
  "errorCode": "INSUFFICIENT_FUNDS",
  "message": "Payment failed: card declined",
  "context": {
    "orderId": "order_456",
    "amount": 99.99,
    "currency": "USD",
    "provider": "stripe"
  },
  "duration_ms": 145,
  "metadata": {
    "environment": "production",
    "version": "2.1.0"
  }
}
```

### Essential Fields

| Field | Purpose | Example |
|-------|---------|---------|
| **timestamp** | When it happened | ISO 8601 format |
| **level** | Severity | DEBUG, INFO, WARN, ERROR, FATAL |
| **message** | Human-readable summary | "Payment processing failed" |
| **traceId** | Request correlation | Unique ID across service boundaries |
| **userId** | Who was affected | For user-specific issues |
| **action** | What operation | Create, update, delete, query |
| **status** | Result | success, failure, timeout |
| **context** | Relevant data | Order ID, amount, retries |
| **duration** | Timing data | For performance analysis |

### Log Levels: Use Correctly

```javascript
logger.fatal('Application cannot recover')      // Server won't start
logger.error('Operation failed, but app runs')  // Failed payment, DB timeout
logger.warn('Degraded behavior or anomaly')     // High memory, rate limit approaching
logger.info('Significant event occurred')       // User logged in, job completed
logger.debug('Detailed diagnostic info')        // Variable values (dev only)
logger.trace('Very detailed call paths')        // Function entry/exit (dev only)
```

**Anti-pattern:** Using `error` level for warnings or `info` for debugging floods alerts.

---

## How to Extract Value From Logs

### 1. **Real-time Alerting**
```
Alert Rules:
- ERROR level → instant notification
- WARN level for payment failures → immediate escalation
- Failed login attempts (>5/minute) → security team alert
```

### 2. **Aggregation & Analysis**
```
Questions to ask your logs:
- What % of transactions fail at each stage?
- What's the 95th percentile response time?
- Which users experience the most errors?
- What time of day are errors most common?
```

### 3. **Distributed Tracing**
```
Use traceId to follow a request:
User Click → API Gateway → Service A → Service B → Database
All services log the same traceId, allowing you to reconstruct the entire flow
```

### 4. **Anomaly Detection**
```
Tools (ELK, Datadog, CloudWatch):
- Baseline: normal error rate is 0.1%
- Alert: if error rate > 1% for 5 minutes
- Action: trigger incident response
```

### 5. **Performance Profiling**
```
Extract insights:
- Slow endpoints: filter by duration_ms > threshold
- Failed requests by error code: group and count
- User experience: trace specific user IDs through their session
```

### 6. **Post-Incident Analysis**
```
Incident: "Payment processing down 10 minutes"
Log analysis:
1. Find all errors during timeframe
2. Identify root cause pattern
3. Correlate with deployments/config changes
4. Build timeline for post-mortem
```

---

## Logging Patterns by Use Case

### Pattern 1: Request/Response Logging
```javascript
// Log incoming request
logger.info('Request received', {
  traceId: generateTraceId(),
  method: req.method,
  path: req.path,
  userId: req.user?.id
});

// Log response (in middleware)
logger.info('Request completed', {
  traceId: req.traceId,
  statusCode: res.statusCode,
  duration_ms: Date.now() - req.startTime
});
```

**Value:** Understand request patterns, identify slow endpoints

### Pattern 2: Error Context Logging
```javascript
try {
  await processPayment(order);
} catch (error) {
  logger.error('Payment processing failed', {
    traceId,
    orderId: order.id,
    userId: order.userId,
    errorCode: error.code,
    message: error.message,
    stack: error.stack,  // Include for debugging
    context: {
      amount: order.total,
      paymentMethod: order.payment.type,
      retryCount: retries
    }
  });
  throw error;
}
```

**Value:** Rapid debugging; understand failure patterns

### Pattern 3: Business Metric Logging
```javascript
// Track important milestones
logger.info('Order completed', {
  orderId: order.id,
  userId: order.userId,
  amount: order.total,
  itemCount: order.items.length,
  conversionSource: order.source,
  processingTime_ms: order.createdAt - now()
});
```

**Value:** Business analytics without separate tools; track conversion funnel

### Pattern 4: Security Event Logging
```javascript
// Log all auth-related events
logger.warn('Failed authentication attempt', {
  userId,
  ipAddress: req.ip,
  reason: 'invalid_password',
  attemptCount: attempts,
  timestamp: new Date()
});

// Log successful auth (for audit trail)
logger.info('User authenticated', {
  userId,
  ipAddress: req.ip,
  method: 'password',
  timestamp: new Date()
});
```

**Value:** Compliance, security threat detection

### Pattern 5: Background Job Logging
```javascript
logger.info('Background job started', { jobId, type: 'send_emails' });

logger.info('Job progress', {
  jobId,
  processed: 1500,
  total: 5000,
  percentage: 30
});

logger.info('Background job completed', {
  jobId,
  type: 'send_emails',
  duration_ms: 245000,
  itemsProcessed: 5000,
  failures: 3
});
```

**Value:** Monitor long-running processes, identify bottlenecks

---

## Common Mistakes & How to Avoid Them

### ❌ Mistake 1: Logging Sensitive Data
```javascript
// Bad
logger.info('User login', { userId, password: user.password });

// Good
logger.info('User login', { userId, loginMethod: 'email' });
```

### ❌ Mistake 2: No Trace IDs
```javascript
// Bad - can't correlate across services
logger.error('Failed to fetch user');

// Good - can follow request through system
logger.error('Failed to fetch user', { traceId, userId });
```

### ❌ Mistake 3: Too Much Logging in Loops
```javascript
// Bad - generates millions of logs
for (let i = 0; i < users.length; i++) {
  logger.info('Processing user', { userId: users[i].id });
}

// Good - log summary
logger.info('Processing users completed', {
  count: users.length,
  duration_ms: elapsed
});
```

### ❌ Mistake 4: Inconsistent Log Levels
```javascript
// Bad - inconsistent severity
logger.error('User not found');      // Should be WARN or INFO
logger.info('Database connection failed'); // Should be ERROR

// Good
logger.warn('User not found');       // Expected scenario
logger.error('Database connection failed'); // System failure
```

### ❌ Mistake 5: String Concatenation
```javascript
// Bad - unstructured, hard to query
logger.info('User ' + userId + ' logged in from ' + ipAddress);

// Good - structured, queryable
logger.info('User logged in', { userId, ipAddress });
```

---

## Implementation Tips

### 1. **Use a Structured Logging Library**
- **Node.js/TypeScript:** Winston, Pino, Bunyan
- **Java:** SLF4J with Logback
- **Python:** Structlog, Python's logging with JSON formatter
- **Benefits:** Structured output, filtering, transport flexibility

### 2. **Implement Centralized Logging**
```
Local logs → Shipper → Central Store → Analysis/Alerting
  (file)        (Fluentd)   (ELK Stack)    (Dashboards)
```

### 3. **Use Log Levels Effectively**
- Production: INFO and above (ERROR, WARN, INFO)
- Staging: DEBUG and above
- Development: DEBUG/TRACE as needed

### 4. **Add Correlation/Trace IDs**
Every request should have a unique ID that flows through all services for end-to-end traceability.

### 5. **Monitor Log Volume**
- Track logs-per-second to detect runaway logging
- Set up alerts for unusual patterns
- Periodically review and cleanup verbose logs

### 6. **Retention & Compliance**
- Retain logs based on regulations (GDPR, HIPAA: often 30-90 days)
- Archive older logs for long-term analysis
- Document retention policies

---

## 80/20 Takeaway

Focus on these core logging practices for maximum ROI:

1. **Log the three M's:** Mutations (writes), Milestones (state changes), Mistakes (errors)
2. **Use traceId everywhere** to correlate related events
3. **Include context:** Who, what, when, and relevant business data
4. **Use correct log levels** to avoid alert fatigue
5. **Never log secrets** (passwords, API keys, tokens)
6. **Structure your logs** as JSON for easy querying and analysis
7. **Set up basic alerting** on ERROR level and critical patterns
8. **Review logs regularly** to understand your system's behavior and spot patterns

---

## Further Reading

- **Structured Logging:** Honeycomb's "Observability Engineering"
- **Log Analysis:** ELK Stack documentation, Datadog intro
- **Distributed Tracing:** OpenTelemetry, Jaeger
- **Best Practices:** The Twelve-Factor App (logging section)
