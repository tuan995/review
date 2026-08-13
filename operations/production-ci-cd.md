# Production Operations & CI/CD Interview Notes

## CV-backed context
Your CV mentions production incident resolution, health checks, deployment workflows, CloudWatch monitoring/alerting, Docker, CI/CD, connection-pool tuning, and operational tooling.

## Production readiness
A backend service should have:
- health/readiness checks
- structured logging
- metrics and alerts
- graceful shutdown
- bounded retries/timeouts
- connection-pool limits
- rollback strategy
- dependency failure handling

## Graceful shutdown
Typical flow:
1. Stop accepting new traffic.
2. Mark instance unready.
3. Finish or cancel in-flight requests/jobs safely.
4. Close DB/Redis/queue connections.
5. Exit before deployment timeout.

## Health checks
- Liveness: process is alive.
- Readiness: service can safely receive traffic.
Avoid making liveness depend on every downstream dependency, or transient dependency failure may restart healthy processes unnecessarily.

## Logging and observability
Useful context:
- request/correlation ID
- service/version
- latency/status
- dependency errors
- queue/job ID
Never log secrets, tokens, passwords, or sensitive payloads unnecessarily.

## CI/CD
Typical stages:
commit -> lint/test -> build -> security/static checks -> image/artifact -> deploy -> smoke/health validation -> monitor/rollback

## Deployment strategies
- Rolling: simple, gradual replacement.
- Blue/green: fast rollback, more infrastructure.
- Canary: expose small percentage first, best for risky changes but requires observability/routing support.

## Incident response
1. Assess impact.
2. Stabilize/mitigate.
3. Gather evidence from metrics/logs/traces.
4. Identify root cause.
5. Fix and validate.
6. Add prevention/detection improvements.

## Interview questions
1. Liveness vs readiness?
2. How do you implement graceful shutdown in Node.js?
3. What happens to in-flight jobs during deployment?
4. How do you choose DB connection-pool size?
5. What metrics would you alert on?
6. Rolling vs blue/green vs canary deployment?
7. What makes a rollback safe?
8. How do you debug a production latency spike?
9. How do retries make an outage worse?
10. What is a correlation ID?
11. How do you avoid logging secrets?
12. How do you handle a dependency outage?
13. What should a postmortem contain?
14. How do you validate a deployment before full rollout?
