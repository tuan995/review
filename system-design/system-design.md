# System Design Interview Notes

## 1. Cách bắt đầu một bài System Design

Không nhảy vào vẽ architecture ngay. Đi theo thứ tự:

1. Clarify requirements.
2. Xác định functional requirements.
3. Xác định non-functional requirements: scale, latency, availability, consistency, durability.
4. Ước lượng traffic và storage.
5. Thiết kế high-level architecture.
6. Chọn data model.
7. Xác định bottleneck và trade-off.
8. Deep dive vào phần interviewer quan tâm.

## 2. Load Balancer
Load balancer phân phối traffic tới nhiều application instances.

Lợi ích:
- Horizontal scaling.
- High availability.
- Health check.
- Có thể hỗ trợ TLS termination, routing.

## 3. Horizontal vs Vertical Scaling
Vertical scaling: tăng CPU/RAM cho một máy.
Horizontal scaling: tăng số instance.

Horizontal scale thường phù hợp khi hệ thống cần tăng availability và throughput lớn, nhưng làm bài toán distributed system phức tạp hơn.

## 4. Stateless service
Backend stateless dễ scale ngang hơn. Session/state nên đặt ở shared store như Redis/DB khi phù hợp.

## 5. Cache
Cache giảm latency và giảm tải DB.

Các vị trí cache:
- Browser/client.
- CDN.
- Reverse proxy.
- Application/local cache.
- Distributed cache như Redis.

Trade-off lớn nhất: invalidation và stale data.

## 6. CDN
CDN cache static/content gần user để giảm latency và tải origin.

Phù hợp cho image, video, JS/CSS, file download và một số API/content cacheable.

## 7. Database replication
Replication tạo nhiều bản sao dữ liệu.

Mục tiêu:
- High availability.
- Read scaling.
- Disaster recovery.

Cần hiểu replication lag và read-after-write consistency.

## 8. Database sharding
Sharding chia dữ liệu ngang giữa nhiều node.

Lợi ích: scale storage và write throughput.

Chi phí:
- Cross-shard query khó hơn.
- Rebalancing.
- Distributed transaction phức tạp.
- Shard key cực kỳ quan trọng.

## 9. Message Queue
Queue tách producer và consumer.

Lợi ích:
- Async processing.
- Buffer traffic spike.
- Retry.
- Decoupling.

Ví dụ use case: email, notification, image processing, payment events, log processing.

## 10. Kafka vs RabbitMQ
### Kafka
Phù hợp event streaming, throughput lớn, retention và replay.

### RabbitMQ
Phù hợp message routing/queue truyền thống, work queue và flexible routing.

Không chọn chỉ theo popularity; chọn theo semantics cần thiết.

## 11. At-most-once / At-least-once / Exactly-once
- At-most-once: có thể mất message nhưng không duplicate.
- At-least-once: không muốn mất nhưng có thể duplicate.
- Exactly-once: semantics khó và thường cần phối hợp nhiều layer.

Trong backend thực tế, at-least-once + idempotency thường rất quan trọng.

## 12. Idempotency
Một operation idempotent có thể retry mà không tạo side effect lặp ngoài ý muốn.

Ví dụ payment API dùng idempotency key để tránh charge hai lần.

## 13. Rate Limiting
Các thuật toán phổ biến:
- Fixed window.
- Sliding window.
- Token bucket.
- Leaky bucket.

Mục tiêu: bảo vệ hệ thống, fairness và chống abuse.

## 14. Circuit Breaker
Khi downstream lỗi liên tục, circuit breaker tạm ngừng gửi request để tránh cascading failure.

Các trạng thái thường nhớ:
- Closed.
- Open.
- Half-open.

## 15. Retry
Retry chỉ nên dùng cho lỗi có khả năng transient.

Best practices:
- Exponential backoff.
- Jitter.
- Retry limit.
- Idempotency.
- Không retry mù quáng lỗi permanent.

## 16. CAP theorem
Trong hệ distributed khi có network partition, phải trade-off giữa Consistency và Availability.

CAP không có nghĩa là hệ thống luôn chỉ chọn đúng hai chữ; cần nói trong bối cảnh partition và consistency model thực tế.

## 17. Strong vs Eventual Consistency
Strong consistency: read thấy write mới nhất theo contract mạnh.
Eventual consistency: replicas có thể lệch tạm thời nhưng hội tụ sau.

Chọn theo domain. Balance/payment thường cần semantics chặt hơn feed/analytics.

## 18. Observability
Ba nhóm quan trọng:
- Logs.
- Metrics.
- Traces.

Theo dõi latency, error rate, throughput và saturation.

## 19. SLI / SLO / SLA
- SLI: chỉ số đo được, ví dụ p99 latency.
- SLO: mục tiêu nội bộ, ví dụ 99.9% availability.
- SLA: cam kết với khách hàng và thường gắn consequence.

## 20. Single Point of Failure
Luôn hỏi: nếu component này chết thì sao?

Cân nhắc redundancy cho load balancer, DB, cache, queue và application nodes.

## Câu hỏi phỏng vấn

### Vì sao dùng queue?
Để decouple, xử lý async, absorb traffic spike và retry độc lập.

### Cache có trade-off gì?
Nhanh hơn nhưng phải xử lý invalidation và stale data.

### Replication khác sharding?
Replication tạo bản sao; sharding chia dataset.

### Khi nào dùng Kafka?
Khi cần event stream throughput lớn, retention, replay và nhiều consumer độc lập.

### Tại sao cần idempotency?
Để retry an toàn trong distributed system nơi timeout và duplicate có thể xảy ra.

### Thiết kế system bắt đầu từ đâu?
Requirements và constraints, không bắt đầu bằng công nghệ.

## Cheat sheet
- Requirements trước architecture.
- Scale: stateless + load balancer.
- Cache giảm latency/load nhưng có stale data.
- Replication = copy; sharding = split.
- Queue = async + decouple + buffer.
- Retry phải có backoff/jitter/idempotency.
- Distributed system phải nghĩ tới failure.
- Observability là phần của design, không phải đồ trang trí.
