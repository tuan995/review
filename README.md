# Backend Interview Handbook

Sổ tay ôn phỏng vấn Backend, tập trung vào kiến thức thực tế, trade-off, câu hỏi thường gặp, câu trả lời ngắn gọn và ví dụ dễ nhớ.

## Nội dung

### 1. Database & SQL
- [Index](database/index.md)
- [JOIN](database/join.md)
- [Transaction & ACID](database/transaction-acid.md)
- [Isolation Level, MVCC, Lock & Deadlock](database/isolation-lock-deadlock.md)
- [EXPLAIN & Query Optimization](database/query-optimization.md)

### 2. Redis
- [Redis Interview Notes](redis/redis.md)
  - Cache-aside
  - TTL / invalidation
  - Cache stampede / penetration / avalanche
  - Distributed lock
  - Persistence / eviction
  - Rate limiting

### 3. Node.js
- [Node.js Interview Notes](nodejs/nodejs.md)
  - Event Loop
  - Promise / async-await
  - Streams / backpressure
  - Worker Threads
  - Memory leak
  - Graceful shutdown

### 4. NestJS
- NestJS Interview Notes *(đang hoàn thiện)*
  - Dependency Injection
  - Module / Provider
  - Middleware / Guard / Interceptor / Pipe
  - Exception Filter

### 5. PostgreSQL
- [PostgreSQL Interview Notes](postgresql/postgresql.md)
  - MVCC
  - VACUUM / ANALYZE
  - B-tree / GIN / GiST / BRIN
  - JSONB
  - Connection pool

### 6. MongoDB
- [MongoDB Interview Notes](mongodb/mongodb.md)
  - Embed vs Reference
  - Index / explain
  - Aggregation
  - Replica Set
  - Sharding

### 7. System Design
- [System Design Interview Notes](system-design/system-design.md)
  - Load Balancer
  - Horizontal scaling
  - Cache / CDN
  - Replication / Sharding
  - Queue
  - Kafka / RabbitMQ
  - Rate Limit
  - Circuit Breaker / Retry
  - CAP / Consistency
  - Observability

## Cách học

Mỗi chủ đề nên học theo 4 bước:

1. Hiểu khái niệm.
2. Hiểu trade-off.
3. Xem ví dụ thực tế.
4. Tự trả lời câu hỏi phỏng vấn mà không nhìn đáp án.

Một vòng ôn tốt là:

**Khái niệm -> Vì sao cần -> Khi nào dùng -> Khi nào không dùng -> Trade-off -> Ví dụ -> Câu hỏi phỏng vấn.**

> Mục tiêu: hiểu bản chất thay vì học thuộc câu chữ.
