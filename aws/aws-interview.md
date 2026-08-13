# AWS Interview Notes — CV-aligned

This chapter is split into two levels:

- **CV-backed experience:** services explicitly present in the CV.
- **Interview expansion:** knowledge worth reviewing because interviewers may drill deeper; it is not automatically a claim that it was used in production.

## AWS services present in the CV

- SQS
- S3
- CloudFront
- CloudWatch
- SES
- DynamoDB
- QLDB
- MediaConvert
- CodeCommit

---

# 1. Amazon SQS

## CV-backed experience

The CV states experience with:
- SQS workers/background processing,
- retry and dead-letter handling,
- shared SQS email delivery,
- order/event processing.

## Core model

SQS decouples producers and consumers. A producer sends a message; a consumer receives and processes it asynchronously.

Typical flow:

`API -> SQS -> Worker -> DB/external API`

Failures can be retried; repeatedly failing messages can be moved to a DLQ.

## Interview expansion

### Standard vs FIFO

**Standard queue**
- very high throughput,
- at-least-once delivery,
- ordering is best effort.

**FIFO queue**
- preserves ordering within a message group,
- supports deduplication semantics,
- useful when ordering is a business requirement.

### Visibility Timeout

After a worker receives a message, SQS temporarily hides it from other consumers.

If processing succeeds, delete the message.

If the worker does not delete it before visibility timeout expires, the message can become visible again.

Therefore:

> A worker should be designed assuming a message may be processed more than once.

### Idempotency

A handler should ideally produce the same business result even if the same logical message is delivered multiple times.

Common approaches:
- event/message ID stored as processed,
- unique database constraint,
- idempotency key,
- conditional update/state transition.

### DLQ

A DLQ isolates messages that repeatedly fail so poison messages do not retry forever.

Be ready to explain:
1. max receive count,
2. alerting,
3. inspection/root-cause workflow,
4. replay/redrive strategy.

### Retry danger

Retries can amplify an outage. If a dependency is down and thousands of messages retry aggressively, the dependency can receive even more traffic.

Mitigations include backoff, jitter, bounded retries, concurrency control, and circuit-breaking where appropriate.

## SQS interview questions

1. Why did you choose SQS?
2. SQS vs BullMQ?
3. Standard vs FIFO?
4. What is visibility timeout?
5. What if processing exceeds visibility timeout?
6. Why can duplicate delivery happen?
7. How do you make a consumer idempotent?
8. What is a DLQ?
9. How do you replay DLQ messages safely?
10. Worker crashes after DB commit but before deleting message — what happens?
11. How do you prevent retry storms?
12. How do you choose worker concurrency?
13. What metrics would you monitor for queue health?

---

# 2. Amazon S3

## CV-backed experience

The CV includes S3 for file/document storage, Shopify digital downloads, uploads/viewing, and cloud systems.

## Core model

S3 is object storage. Think in terms of:

`bucket + object key + object data + metadata`

It is not a normal filesystem and should not be modeled like a local disk.

## Interview expansion

### Presigned URL

A backend can generate a temporary signed URL so a client can upload/download directly without exposing AWS credentials.

Typical upload architecture:

`Client -> API (request permission) -> presigned URL -> Client uploads directly to S3`

Benefit: large file bytes do not need to pass through the application server.

### Security review

Know:
- IAM least privilege,
- private buckets,
- bucket policies,
- object permissions,
- encryption,
- expiration of temporary access.

## S3 interview questions

1. Why upload directly to S3 instead of through Node.js?
2. What is a presigned URL?
3. How do you prevent unauthorized downloads?
4. How do you validate uploaded file type/size?
5. How do you handle multipart upload?
6. What happens to abandoned uploads?
7. How would you design secure digital-download delivery?

---

# 3. CloudFront

## CV-backed experience

CloudFront appears in the Shopify SaaS platform stack.

The CV does not specify the exact CloudFront configuration used, so treat details below as interview expansion unless they match your actual work.

## Interview expansion

CloudFront is a CDN that caches/distributes content closer to users.

Common architecture:

`User -> CloudFront -> S3/origin`

Important concepts:
- cache key,
- TTL,
- cache invalidation,
- origin,
- signed URL/cookie for protected content.

## Questions

1. Why put CloudFront in front of S3?
2. Cache hit vs cache miss?
3. How do you invalidate stale content?
4. How would you serve private files through CloudFront?
5. What should and should not be part of a cache key?

---

# 4. CloudWatch

## CV-backed experience

The CV states CloudWatch monitoring/alerting in the BlueSky cloud migration.

## Interview expansion

Know the distinction between:
- metrics,
- logs,
- alarms,
- dashboards.

For an async backend, useful signals include:
- error rate,
- latency,
- queue depth,
- oldest message age,
- DLQ count,
- worker throughput,
- DB connection saturation.

## Questions

1. What did you monitor in CloudWatch?
2. Which metrics indicate an SQS consumer is falling behind?
3. What alert is actionable rather than noisy?
4. Logs vs metrics — when do you use each?
5. How would you investigate a sudden latency increase?

---

# 5. SES

## CV-backed experience

The CV includes SES for digital-download/email workflows.

## Interview expansion

Topics to review:
- transactional email,
- bounce/complaint handling,
- retries,
- asynchronous sending,
- idempotency to avoid duplicate emails.

## Questions

1. Why send email asynchronously?
2. How do you avoid duplicate email after retry?
3. How do you handle bounce/complaint events?
4. What if SES is temporarily unavailable?

---

# 6. DynamoDB

## CV-backed experience

The CV states DynamoDB was used for operational data in the BlueSky cloud migration and in earlier work.

## Interview expansion

DynamoDB design starts from access patterns rather than relational normalization.

### Partition key

The partition key determines data distribution. A poor key can create a hot partition.

### Composite key

A partition key plus sort key supports multiple ordered items under the same logical partition.

### Conditional writes

Useful for concurrency control and safe state transitions.

## Questions

1. How did you choose partition key?
2. What is a hot partition?
3. Partition key vs sort key?
4. Query vs Scan?
5. GSI vs LSI?
6. How do conditional writes help concurrency?
7. How would you model an access pattern before creating a table?
8. DynamoDB vs PostgreSQL — when would you choose each?

---

# 7. QLDB

## CV-backed experience

The CV states QLDB was used for immutable transaction records during a cloud migration.

## Interview preparation

Be ready to explain the actual reason it was selected in that project, especially the requirement around immutable/auditable history.

Questions:
1. Why was an immutable transaction history needed?
2. Why not store history in a normal relational table?
3. What business requirement drove QLDB?
4. How was operational data separated from transaction history?

Do not invent QLDB internals you did not personally work with.

---

# 8. MediaConvert

## CV-backed experience

MediaConvert appears in the Digital Downloads Shopify App project.

## Interview preparation

Be ready to explain:
- what media needed conversion,
- how a conversion job was triggered,
- where input/output lived,
- how completion/failure was detected.

The CV does not specify these implementation details, so fill them only from real project memory.

---

# 9. AWS architecture questions based on the CV

## Design question A — Reliable Shopify order processing

Design:

`Shopify webhook -> API -> queue -> workers -> operational DB -> analytics pipeline`

Discuss:
- webhook authentication,
- quick acknowledgement,
- queueing,
- duplicate events,
- retries,
- DLQ,
- idempotency,
- distributed locking,
- observability,
- replay.

## Design question B — Digital download platform

Discuss:
- upload to S3,
- authorization,
- secure download,
- CDN,
- email notification,
- media processing,
- expiration/revocation,
- audit trail.

## Design question C — Background synchronization

Discuss:
- SQS vs BullMQ,
- concurrency,
- per-tenant fairness,
- rate limits,
- retries,
- idempotency,
- poison jobs,
- monitoring.

---

# Rapid-fire AWS checklist

Before a Senior Backend interview, be able to explain clearly:

- SQS visibility timeout
- at-least-once delivery
- idempotent consumer
- DLQ/redrive
- retry + exponential backoff + jitter
- S3 presigned URL
- S3 authorization/IAM basics
- CloudFront caching basics
- CloudWatch metrics/logs/alarms
- SES async email failure handling
- DynamoDB partition/sort key
- hot partition
- Query vs Scan
- conditional writes
- why operational and immutable/audit data may use different stores

# Key interview principle

When discussing AWS, distinguish three things explicitly:

1. **What I personally implemented.**
2. **What my team/system used.**
3. **What I know conceptually but did not directly implement.**

That distinction makes a Senior-level answer more credible than pretending every technology in the stack was personally designed end-to-end.
