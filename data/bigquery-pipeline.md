# BigQuery & Data Pipeline Interview Notes

## CV-backed context
Your CV states that you worked on event ingestion, PostgreSQL operational storage, batched BigQuery delivery, reporting queries, schema validation/fallbacks, and retention-related work.

## OLTP vs analytics
Operational databases optimize transactional reads/writes. BigQuery is designed for large analytical scans and aggregations.

A common architecture:

producer -> queue/worker -> operational DB -> batch/export -> BigQuery -> reports

## Batch vs streaming
### Batch
- simpler and cheaper in many cases
- higher latency
- easier retry/reconciliation

### Streaming
- lower latency
- more operational complexity
- duplicates and ordering become more important

## Idempotency and duplicates
Event pipelines often provide at-least-once delivery. Use stable event IDs, deduplication keys, or merge/upsert logic where appropriate.

## Schema evolution
Good strategies:
- validate input schemas
- prefer backward-compatible additions
- version breaking changes
- isolate malformed events instead of blocking the full pipeline

## Failure handling
- bounded retries with backoff
- dead-letter/quarantine path
- metrics for lag/error rate
- reconciliation jobs for gaps
- preserve enough metadata to replay safely

## Interview questions
1. Why copy data from PostgreSQL to BigQuery?
2. Batch vs streaming: when would you use each?
3. How do you handle duplicate events?
4. How do you recover if BigQuery delivery fails for several hours?
5. How do you evolve event schemas safely?
6. How do you prevent one malformed event from blocking a batch?
7. What metrics would you monitor in a data pipeline?
8. How do you detect missing events?
9. Why keep an operational database if BigQuery contains the data?
10. How do you backfill historical data safely?
11. How do you control BigQuery query cost?
12. What is partitioning and why does it matter?
13. What is clustering in BigQuery?
14. How would you design retention and archival?

## Senior answer pattern
Describe source of truth, delivery guarantees, idempotency, retry/replay strategy, observability, and how you verify data completeness.
