# SQS Connection Pool Analysis and Configuration

[English](10_sqs_connection_pool_analysis_and_configuration.md) | [繁體中文](../zh-tw/10_sqs_connection_pool_analysis_and_configuration.md) | [日本語](../ja/10_sqs_connection_pool_analysis_and_configuration.md) | [Back to Index](../README.md)

---

## Background
- Experiment Date: 2025/12/09
- Difficulty: 🤬🤬🤬
- Description: After adding a fourth SQS queue in the Production environment, a connection pool exhaustion error occurred. The error message indicates that a connection could not be acquired from the pool within the configured maximum time.

## Symptoms

- Kibana log:
  ```
  Unable to execute HTTP request: Acquire operation took longer than the configured maximum time.
  This indicates that a request cannot get a connection from the pool within the specified maximum time.
  ```
- The error occurred in `QueueAsyncClientImpl` when processing non-transactional EMAIL messages (directly calling AWS SDK V2 at the underlying layer)
- The service could not properly process messages from the new queue

## Debugging Process

### Phase 1: Verify Infrastructure Configuration

The following items were confirmed to be correctly configured:

| Item | Status | Description |
|------|--------|-------------|
| SQS Queues | ✅ | Originally had 3 queues, added a fourth queue |
| Poller Configuration | ✅ | Polls messages every 1 second (`sqs.communication.poller.schedule: "1s"`), this is a custom property defined in the application |
| Batch Size | ✅ | Transactional: 10, Non-transactional: 10, which means how many messages to fetch from SQS each time, max = 10 |
| SQS Client | ⚠️ | Using AWS SDK default configuration (this is where the problem lies) |

### Phase 2: Analyze Queue Operation Patterns

Examined the operation patterns of each queue:

**Original 3 Queues:**
- `transactionalQueueUrl`: Transactional message queue
- `queueUrl`: Non-transactional SMS message queue  
- `emailMetricsQueueUrl`: Email metrics queue

**New Fourth Queue:**
- Non-transactional EMAIL message queue
- Each poller execution:
  - 1 `receiveMessage` call (batch read up to 10 messages)
  - 10 `deleteMessage` calls (one per message)
  - **Total: 11 HTTP connections required**

**Conclusion**: The new queue's operation pattern is connection-intensive, requiring 11 connections per execution.

### Phase 3: Connection Pool Capacity Analysis

Examined the SQS Client connection pool configuration:

| Configuration Item | Default Value | Actual Requirement |
|-------------------|---------------|---------------------|
| `max-concurrency` | 50 | Insufficient |
| `connection-acquisition-timeout-seconds` | 2 seconds | Too short |
| `max-pending-connection-acquires` | 10,000 | Sufficient |

**Findings**:
- Default connection pool only has 50 connections
- The new queue requires 11 connections per second, and combined with traffic from other queues, easily exceeds 50 connections
- Connection acquisition timeout is only 2 seconds, which is insufficient to wait for connection release in high traffic scenarios

## Root Cause

### New Queue Addition Leading to Connection Pool Exhaustion

**Key Finding**: When adding the fourth queue, this queue adopted a connection-intensive operation pattern (requiring 1 receiveMessage + 10 deleteMessage = 11 connections per execution), significantly increasing concurrent HTTP connection requirements and exceeding the default connection pool capacity of 50.

### Architecture Changes

**Before (3 Queues):**
- `transactionalQueueUrl`: Transactional message queue
- `queueUrl`: Non-transactional SMS message queue  
- `emailMetricsQueueUrl`: Email metrics queue
- Each poller execution processes messages from these queues

**After (4 Queues - After Adding New Queue):**
- `transactionalQueueUrl`: Transactional message queue
- `queueUrl`: Non-transactional SMS message queue  
- `emailMetricsQueueUrl`: Email metrics queue
- **New queue**: Non-transactional EMAIL message queue (newly added)
  - Each poller execution:
    - 1 `receiveMessage` call (batch read up to 10 messages)
    - 10 `deleteMessage` calls (one per message)
    - **Total: 11 HTTP connections required**

### Impact

- **Connection-intensive pattern**: The new queue requires 11 connections per poller execution (1 receiveMessage + 10 deleteMessage)
- **Increased connection pool pressure**: With poller running every 1 second, this adds significant concurrent connection demand
- **Connection pool exhaustion**: Default connection pool (50 connections) is insufficient when combined with traffic from other queues
- **Connection acquisition timeout**: New API calls cannot acquire connections in time when the connection pool is exhausted

## Problem Analysis

### Connection Pool Operation Mechanism

1. **Connection Pool Location**:
   - The connection pool is **not on the SQS service side**, but **on the application side** (within the application)
   - Managed by `NettyNioAsyncHttpClient`, because we are using the Quarkus framework
   - Located in application memory
   - Used to reuse HTTP connections

2. **Connection Pool Operation Flow**:
   ```java
   // When you call
   sqsAsyncClient.receiveMessage(...)
   // SDK internally will:
   // 1. Acquire an HTTP connection from the connection pool
   // 2. Use this connection to send HTTP request to SQS
   // 3. Receive response
   // 4. Return connection to pool (for next use)
   ```

3. **Why is a Connection Pool Needed?**
   - Each SQS API call requires establishing an HTTP connection
   - Connection establishment has overhead (TCP handshake, TLS handshake)
   - Connection pool can reuse connections, improving performance

### Why Connection Pool Exhaustion Occurs

- When all connections are in use (e.g., processing other API calls)
- New receiveMessage calls need to wait for connection release
- If wait time exceeds `connectionAcquisitionTimeout` (default 2 seconds), timeout occurs

### New Queue's Connection Requirements

**Normal Traffic Analysis:**

| Queue | receiveMessage | deleteMessage | Total Connections/Execution |
|-------|----------------|---------------|----------------------------|
| Queue 1 - Transactional | 1 call | Up to 10 calls | Up to 11 |
| Queue 2 - Non-transactional SMS | 1 call | Up to 10 calls | Up to 11 |
| Queue 3 - Email metrics | Depends on actual usage | Depends on actual usage | Depends on actual usage |
| **Queue 4 - New EMAIL queue** | **1 call** | **10 calls** | **11** |

**Connection Pool Demand:**
- **Before adding new queue**: Existing 3 queues with their respective traffic patterns
- **After adding new queue**: New queue adds 11 connections per second
- **Total concurrent connections**: Can easily exceed 50 (default connection pool size) during normal operations
- **Burst scenarios**: Connection demand can spike significantly

**Burst Traffic Estimation:**
- Burst traffic can be 10x normal traffic
- The new queue alone can generate 110 connections/second during burst (11 connections × 10x)
- Combined with other queues' traffic, total concurrent connection demand can exceed 200

## Solution

### Create Custom SQS Client Producer

Created a custom SQS client Producer (`AwsSqsClientProducer`) with appropriate connection pool parameters.

### Production Environment Configuration

```yaml
aws:
  sqs:
    max-concurrency: 200                    # Handle 200 concurrent requests
    max-pending-connection-acquires: 2000   # Allow maximum 2000 requests to queue
    connection-timeout-seconds: 3           # Connection establishment timeout 3 seconds
    connection-acquisition-timeout-seconds: 10  # Connection acquisition timeout 10 seconds
    api-call-timeout-seconds: 15            # API call total timeout 15 seconds
    api-call-attempt-timeout-seconds: 15    # API call attempt timeout 15 seconds
```

- Note: If `max-pending-connection-acquires` is not explicitly specified, it will cause a runtime exception. After calculation, we found the default value of 10,000 is too high, so we lowered it.

### Configuration Parameter Description

| Parameter | Default Value | Recommended Value | Description |
|-----------|---------------|-------------------|-------------|
| max-concurrency | 50 | 200 | Maximum concurrent requests |
| max-pending-connection-acquires | 10,000 | 2,000 | Maximum queued requests |
| connection-acquisition-timeout-seconds | 2 | 10 | Connection acquisition timeout (seconds) |
| connection-timeout-seconds | 10 | 3 | Connection establishment timeout (seconds) |
| api-call-timeout-seconds | Unlimited | 15 | API call total timeout (seconds) |
| api-call-attempt-timeout-seconds | Unlimited | 15 | API call attempt timeout (seconds) |

### Configuration Parameter Calculation Logic

**maxConcurrency (200)**:
- Purpose: Sets the maximum number of concurrent requests
- For HTTP/1.1: Equivalent to maximum number of connections
- For HTTP/2: The number of connections depends on the maximum streams allowed per connection
- Recommended value: 200 (based on actual traffic requirements)

**maxPendingConnectionAcquires (2000)**:
- Purpose: Maximum number of requests allowed to queue when maxConcurrency is reached
- Calculation logic:
  - Normal case: Multiple queues generating concurrent requests, including the new queue's 11 connections/second
  - Burst case: Can easily exceed 200 connections/second
  - If processing time is 0.5 seconds, there will be 100+ new requests within 0.5 seconds during burst
  - If all 200 connections are in use, a maximum of 100+ requests need to queue
  - With a safety margin: Recommended `2000`
- Recommended value: `2000` (can handle bursts without excessive resource usage)

**connectionAcquisitionTimeout (10 seconds)**:
- Purpose: Timeout for acquiring a connection from the connection pool
- Issue: In high traffic scenarios, 2 seconds may not be enough to wait for connection release
- Recommended value: 10 seconds (gives requests more time to wait for connections)

## Why QA Environment Has No Issues?

### QA Environment Configuration Differences

| Configuration Item | Production | QA |
|-------------------|------------|-----|
| Poller frequency | 1 second | 3 seconds |
| Transactional batch size | 10 | 10 |
| Non-transactional batch size | 10 | 5 |
| New queue connection demand | 11 connections/second | Lower (due to longer poller interval) |
| Request rate | Higher (multiple queues + new queue) | ~1/3 of Production |

### Analysis

- QA environment request rate is significantly lower than Production (approximately 1/3)
- Longer poller interval (3 seconds) reduces connection demand per second
- Even with the default connection pool (50 connections), it's sufficient to handle QA traffic
- The longer poller interval provides more time to release connections between executions
- Due to the above differences, QA environment's load testing could not detect this issue, and Production traffic is almost continuous 24/7. Even during non-peak hours, there is still significant information flow, which is why using default settings caused immediate errors after deployment

## Key Learnings

### Importance of Connection Pool Location

| Characteristic | Description |
|----------------|-------------|
| Location | **Application side**, not on SQS service side |
| Management | Managed by NettyNioAsyncHttpClient |
| Storage | In application memory |
| Purpose | Reuse HTTP connections |

### Connection Pool Limits

The connection pool has an upper limit, which is `maxConcurrency`:
- Connection pool limit = `maxConcurrency` (currently 200)
- When the limit is reached, new requests will queue and wait
- Queue limit = `maxPendingConnectionAcquires` (currently 2000)
- Wait time limit = `connectionAcquisitionTimeout` (currently 10 seconds)

### Architecture Diagram

```
Communication service (ECS Container)
├── AwsSqsClientProducer
│   └── SqsAsyncClient
│       └── NettyNioAsyncHttpClient  ← Connection Pool is here!
│           └── HTTP Connection Pool (maxConcurrency: 200)
│               ├── Connection 1 ──┐
│               ├── Connection 2 ──┤
│               ├── Connection 3 ──┤
│               └── ...            │
│                                  │
└── When calling SQS API           │
    ├── receiveMessage() ──────────┼──> Requires an HTTP connection
    ├── deleteMessage() ───────────┼──> Requires an HTTP connection
    └── sendMessage() ─────────────┼──> Requires an HTTP connection
                                   │
                                   ▼
                           HTTP/HTTPS connection to AWS
                                   │
                                   ▼
                            SQS Service (AWS)
```

## Prevention Measures

1. **Evaluate Connection Requirements When Adding Queues**
   - Calculate connection requirements for each queue (receiveMessage + deleteMessage count)
   - Assess whether total concurrent connection requirements exceed existing connection pool capacity

2. **Monitor Connection Pool Usage**
   - Monitor connection pool usage rate
   - Set alerts when connection pool usage exceeds threshold

3. **Adjust Connection Pool Configuration in Advance**
   - Adjust connection pool capacity before adding high-traffic queues
   - Consider burst traffic requirements

4. **Test Environment vs Production Environment Configuration Differences**
   - Understand configuration differences between test and production environments
   - Issues may not appear in test environments but will in production, and load testing may not be able to simulate them

## Conclusion

- This debugging experience reminds us that in AWS SQS applications, the connection pool is an easily overlooked but critical component. Especially when adding queues, we must carefully evaluate each queue's connection requirement patterns to ensure services can operate normally.

- Production environments should apply customized settings after careful calculation, and should not use default values directly.

- **Remember**: The connection pool is on the application side, not on the service side. When adding queues, especially those with connection-intensive operation patterns, we must assess whether the connection pool capacity is sufficient!

## A Restaurant Analogy for Non-Technical Readers

You can think of the HTTP connection pool as a restaurant.

### Original Situation

- The restaurant originally had only 50 seats.
- Each SQS API request is like a guest who wants to dine at the restaurant.
- Guests must acquire a seat before they can start dining.
- After finishing their meal, the seat is released for the next guest.

### After Adding a New Queue

Adding a fourth queue is like opening a new order entry point.

Therefore:

- More guests come to the restaurant at the same time.
- The original 50 seats quickly fill up.
- Later guests can only wait in the waiting area.
- If guests wait too long and still cannot get a seat, they give up and leave.

This "leaving after waiting too long" is equivalent to the system throwing:

```text
Acquire operation took longer than the configured maximum time.
```

---
## Difference Between connectionAcquisitionTimeout and connectionTimeout

These two settings are easily confused.

### connectionAcquisitionTimeout

Represents:
> How long a request is willing to wait at most to acquire an available connection from the connection pool.

Restaurant analogy:
> How long a guest is willing to wait at the door for a seat.

This is the stage where the error occurred in this incident.

### connectionTimeout

Represents:
> After acquiring a connection resource, the maximum time allowed to wait when actually establishing a TCP/TLS connection with the remote service.

Restaurant analogy:
> After a guest has already gotten a seat, how long staff can spend at most establishing contact with the kitchen.

Therefore:

```text
connectionAcquisitionTimeout = 10 seconds
connectionTimeout = 3 seconds
```

They should not be confused with each other.

---

## References
- [Configure the Netty-based HTTP client](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/http-configuration-netty.html) 
- [NettyNioAsyncHttpClient.Builder](https://sdk.amazonaws.com/java/api/latest/software/amazon/awssdk/http/nio/netty/NettyNioAsyncHttpClient.Builder.html)
