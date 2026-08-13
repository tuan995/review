# Bull / BullMQ & Background Jobs Interview Notes

## CV-backed context
Your CV states that you built and tuned Bull/BullMQ workers, used Redis/Bull for background processing, and implemented retries, DLQ-style handling, distributed locks, and recovery mechanisms. These notes focus on those areas.

## Core concepts
- A background job moves slow or retryable work out of the request/response path.
- Bull/BullMQ uses Redis as the backing store.
- A job should be idempotent whenever retries are possible.
- Concurrency controls how many jobs one worker processes in parallel.
- Retries should use bounded attempts and usually backoff.
- Failed jobs need observability and an operational recovery path.

## Common failure cases
- Worker crashes after side effect but before ACK/update -> duplicate effect unless idempotent.
- Redis/network instability -> delayed processing or repeated attempts.
- Too much concurrency -> DB/API saturation.
- Long-running jobs -> lock timeout or stalled-job recovery concerns.
- Poison jobs -> endless retry loops unless bounded.

## Interview questions
1. Why move work to BullMQ instead of handling it synchronously?
2. How do you make a BullMQ job idempotent?
3. What happens if a worker crashes after partially completing a job?
4. How do retries and backoff work?
5. How do you prevent retry storms?
6. How do you choose worker concurrency?
7. How would you detect and recover stalled jobs?
8. How do you prevent two workers from processing the same logical task?
9. What is the role of Redis locks in your design?
10. When would you use SQS instead of BullMQ?
11. How do you handle poison jobs?
12. How do you monitor queue lag, failures, and retry count?

## Senior follow-ups
- Exactly-once is usually not guaranteed; design for at-least-once delivery plus idempotency.
- Keep side effects transactional where possible, or persist an idempotency key/status before external calls.
- Concurrency should be tuned against downstream capacity, not just CPU.
- Retry only transient failures; validation/business-rule failures should normally fail fast.
