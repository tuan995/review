# 13 — System Design Speaking Framework

Mục tiêu của chương này không phải học thuộc một architecture mà là **biết nói suy nghĩ của mình thành từng bước**.

# Framework

## 1. Clarify requirements

> Trước khi thiết kế em muốn xác nhận functional requirements, expected traffic/data size, consistency requirement và phần nào quan trọng nhất: latency, availability hay correctness.

Đừng thiết kế cho “millions of users” nếu interviewer chưa đưa scale.

---

## 2. High-level flow

> Em bắt đầu từ client → API → service → database, sau đó mới thêm cache, queue hoặc search nếu requirement thực sự cần.

```text
Client
  |
Load Balancer
  |
API instances
  |
  +---- Redis
  |
Database
  |
Queue → Workers → External APIs
```

---

## 3. Database

Khi chọn DB, nói **tại sao**:

> Nếu dữ liệu có relation rõ và cần transaction mạnh, em ưu tiên relational DB. Nếu access pattern/document model phù hợp hơn với MongoDB thì em cân nhắc document DB. Em không chọn database chỉ vì quen dùng.

### Index?

> Thiết kế index từ access pattern, không index tất cả column.

---

## 4. Cache

> Em chỉ thêm Redis khi có read hot path/costly computation/external calls cần giảm. Sau đó phải trả lời TTL và invalidation.

---

## 5. Queue

> Queue dùng để decouple task dài, absorb traffic burst, retry và control worker concurrency. Nhưng queue mang lại eventual consistency và cần idempotency/monitoring.

---

## 6. External API

> Với Shopify/Stripe/Google em coi external service là unreliable dependency: timeout, rate limit, retry, circuit behavior, idempotency và reconciliation cần được nghĩ tới.

---

## 7. Scale

> Khi traffic tăng, trước tiên em đo bottleneck. API stateless có thể horizontal scale sau load balancer. Database có thể cần index/query optimization, connection control, read replica hoặc partitioning tùy bottleneck. Không nhảy ngay tới sharding.

---

## 8. Reliability

Luôn hỏi:

- Nếu worker chết thì job có mất không?
- Nếu webhook gửi hai lần?
- Nếu DB update thành công nhưng external call fail?
- Nếu Redis down?
- Nếu API trả 429?
- Nếu deploy nhiều instance thì cron có duplicate?

---

# Ví dụ: thiết kế Shopify synchronization service

## Bài nói

> Shopify là source of truth. Em lưu local data cần cho application trong database để giảm latency và API calls.
>
> Với thay đổi cần gần realtime em nhận webhook. Webhook handler verify request, deduplicate và enqueue processing nếu task dài. Worker update database với concurrency limit để tránh overload DB và Shopify API.
>
> Vì webhook có thể miss hoặc fail, em có scheduled reconciliation job để định kỳ so sánh/sync lại dữ liệu. Với Shopify rate limit, worker có limiter và retry/backoff.
>
> Metrics quan trọng gồm webhook failure, queue depth, sync latency, 429 rate và reconciliation mismatch.

### Interviewer: tại sao vừa webhook vừa cron?

> Webhook tối ưu freshness, reconciliation tối ưu correctness lâu dài.

### Nếu có 100k shops?

> Em partition workload và queue jobs thay vì một cron process toàn bộ. Worker scale horizontally nhưng concurrency/rate phải được giới hạn theo downstream capacity và API policy.

### Exactly-once processing?

> Trong distributed system em thường thiết kế theo at-least-once delivery cộng idempotent processing thay vì phụ thuộc vào exactly-once tuyệt đối.

---

# Checklist nói System Design

`Requirements → scale → API → data model → DB → cache → queue → failure → security → observability → bottleneck/trade-off`

## Câu kết tốt

> Đây là high-level design ban đầu. Nếu cần đi sâu em sẽ chọn một bottleneck hoặc consistency flow cụ thể để phân tích tiếp thay vì thêm component không có requirement.
