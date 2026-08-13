# STAR Interview Preparation

These stories are based on claims in the CV. Add exact metrics and implementation details only from real project memory.

## Senior answer structure

Use STAR, but spend most of the answer on **Action**:

`Situation -> Task -> Action -> Result -> Trade-off -> What I would improve`

For technical stories also cover failure cases and why you chose the design.

## 1. SQS order/event processing

**Situation:** Reporting/event services processed Shopify order and usage events asynchronously.

**Task:** Make background processing reliable under transient failures and duplicate/retried work.

**Action supported by CV:** built SQS workers; added retry/DLQ paths; used Redis locks; added indexes and change-stream gap recovery.

**Result:** describe the real reliability/operational impact. The CV does not provide numeric metrics, so do not invent them.

Follow-ups:
- Why SQS?
- How did retry/DLQ work?
- How did you make processing idempotent?
- Worker crashes after DB write but before acknowledging the message — what happens?
- What metrics did you monitor?

## 2. Shopify token migration

**Situation:** The platform needed a safe path toward expiring offline access tokens.

**Task:** Migrate without breaking existing stores or doing a risky all-at-once rollout.

**Action supported by CV:** built an HMAC-protected token provider and feature-flagged migration path; hardened Shopify connection retries.

**Result:** reduced rollout risk. Add concrete impact only if you remember it accurately.

Follow-ups:
- Why feature flags?
- How would rollback work?
- HMAC vs encryption?
- How did you avoid retry storms?

## 3. Event pipeline to BigQuery

**Situation:** Order/usage events powered operational reports and analytics.

**Task:** Validate, persist and deliver events efficiently across different data workloads.

**Action supported by CV:** schema validation/fallbacks; PostgreSQL/TimescaleDB operational storage; batched BigQuery delivery; reporting queries; connection-pool tuning.

Follow-ups:
- Why not send every event synchronously to BigQuery?
- Why separate operational and analytical stores?
- How did you handle batch failure and duplicates?
- What happened when BigQuery was unavailable?

## 4. MongoDB change-stream recovery

**Situation:** A change-stream consumer could disconnect or fail and risk missing data changes.

**Task:** Make the pipeline recoverable instead of silently losing changes.

**Action supported by CV:** implemented change-stream gap recovery.

Prepare the exact mechanism from memory: resume tokens/checkpoints/reconciliation or whatever the project actually used.

Follow-ups:
- What is a change stream?
- What happens after disconnect?
- How do you detect and recover a gap?
- How do you avoid duplicates during replay?

## 5. BullMQ synchronization

**Situation:** Product, discount and metafield synchronization ran as background work.

**Task:** Keep synchronization reliable without blocking request processing.

**Action supported by CV:** built/tuned BullMQ workers and consolidated background jobs into the main application.

Follow-ups:
- BullMQ vs SQS?
- How did you choose concurrency?
- What happens to stalled jobs?
- How do retries interact with non-idempotent operations?

## 6. Production race condition

**Situation:** The CV mentions production incident resolution and a dashboard loading race condition.

Prepare the real debugging story:

`symptom -> evidence/logs -> reproduction -> root cause -> fix -> regression prevention`

Follow-ups:
- Why was it a race condition?
- Why did tests miss it?
- How did you prove the fix worked?
- What monitoring did you add?

## 7. Cloud migration

**Situation:** A cloud-native backend replaced legacy Railsbank services in Singapore.

**Task:** Build modules for operational data and immutable transaction records.

**Action supported by CV:** DynamoDB for operational data, QLDB for immutable transaction records, CloudWatch monitoring/alerting, CodeCommit/TeamCity CI/CD.

Follow-ups:
- Why DynamoDB?
- How did you choose keys?
- Why immutable records?
- What did CloudWatch monitor?

# Behavioral questions

1. Hardest production issue you debugged?
2. Migration where you minimized rollout risk?
3. Technical decision you would change today?
4. Race condition/concurrency bug you fixed?
5. Performance bottleneck you diagnosed?
6. System you made more observable?
7. Failure mode discovered under production traffic?
8. Trade-off between consistency, reliability and performance?

# Rule for every story

Know these without hesitation:

1. Why the problem mattered.
2. Why you chose the design.
3. Biggest failure mode.
4. Biggest trade-off.
5. Concrete result you can defend.
