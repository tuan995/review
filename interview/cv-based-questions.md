# CV-Based Senior Backend Interview Questions

> Bộ câu hỏi được thiết kế dựa trên kinh nghiệm và công nghệ được nêu trong CV. Mục tiêu là chuẩn bị cho kiểu câu hỏi interviewer đào sâu từ chính những gì ứng viên nói mình đã làm.

## Cách trả lời

Với câu hỏi về project, ưu tiên cấu trúc:

1. Context — hệ thống giải quyết vấn đề gì?
2. Problem — vấn đề kỹ thuật cụ thể là gì?
3. Decision — bạn chọn giải pháp nào và tại sao?
4. Trade-off — giải pháp đánh đổi điều gì?
5. Failure handling — nếu dependency/job/process fail thì sao?
6. Result — kết quả hoặc tác động thực tế.

Không nên chỉ kể tên công nghệ.

---

## 1. Senior Backend / Architecture

### Q1. Hãy mô tả architecture của một Shopify SaaS platform bạn từng làm.

Interviewer có thể đào sâu:
- Merchant backend và storefront public API khác nhau thế nào?
- Background worker nằm ở đâu?
- Redis/SQS được dùng vào việc gì?
- Service nào là source of truth?
- Khi Shopify API unavailable thì hệ thống xử lý thế nào?

### Q2. Khi nào bạn tách background processing khỏi HTTP request?

Ý cần nói:
- request không nên chờ tác vụ lâu
- retry được
- traffic spike có thể buffer qua queue
- scale worker độc lập
- đổi lại phải xử lý eventual consistency, duplicate message và observability

### Q3. Nếu hệ thống đang chạy tốt, tại sao vẫn cần migration theo feature flag?

Follow-up:
- rollout từng phần thế nào?
- rollback ra sao?
- old/new format coexist thế nào?
- metrics nào quyết định tiếp tục rollout?

---

## 2. Node.js / TypeScript

### Q4. Node.js single-threaded nhưng tại sao vẫn xử lý nhiều request đồng thời?

Nói về event loop, async I/O và thread pool/runtime facilities.

### Q5. CPU-bound task ảnh hưởng Node.js server thế nào?

Follow-up:
- worker_threads khi nào phù hợp?
- queue worker khác worker_threads thế nào?

### Q6. Promise.all với hàng nghìn DB/API calls có vấn đề gì?

Nói về connection pool, rate limit, memory và bounded concurrency.

### Q7. Một Node.js process tăng memory liên tục. Bạn debug thế nào?

Gợi ý:
- metrics
- heap snapshot
- retained references
- timers/listeners/cache không giới hạn
- reproduce trước khi sửa

---

## 3. Queue — AWS SQS / Bull / BullMQ

### Q8. Tại sao dùng SQS thay vì gọi service trực tiếp?

Follow-up:
- latency đổi lấy resilience thế nào?
- message duplication xử lý ra sao?

### Q9. Retry nên thiết kế như thế nào?

Cần biết:
- retry transient error
- exponential backoff + jitter
- max attempts
- permanent error không retry vô hạn

### Q10. DLQ là gì và tại sao cần DLQ?

Câu Senior follow-up:
> Message vào DLQ rồi thì ai xử lý và xử lý thế nào?

Nên nói về alert, inspect root cause, fix, replay/redrive và idempotency.

### Q11. SQS có đảm bảo exactly-once processing không?

Điểm quan trọng: application nên thiết kế consumer idempotent thay vì giả định message chỉ tới một lần.

### Q12. Làm thế nào tránh xử lý cùng một job hai lần?

Có thể nói về:
- idempotency key
- unique constraint
- Redis lock
- DB state transition

### Q13. BullMQ và SQS khác nhau thế nào?

So sánh:
- Redis-backed vs managed queue
- operational responsibility
- durability
- delayed/repeatable jobs
- scaling và failure model

---

## 4. Redis / Distributed Lock

### Q14. Bạn dùng Redis lock để giải quyết vấn đề gì?

Interviewer muốn nghe một race condition cụ thể, không chỉ định nghĩa distributed lock.

### Q15. Nếu worker lấy lock rồi crash thì sao?

Phải nhắc TTL.

### Q16. Nếu task chạy lâu hơn TTL của lock thì sao?

Follow-up mạnh:
- lock renewal
- fencing token
- idempotency
- thiết kế critical section nhỏ

### Q17. Cache stampede là gì? Bạn xử lý thế nào?

Có thể dùng lock, stale-while-revalidate, TTL jitter hoặc request coalescing.

---

## 5. PostgreSQL / TimescaleDB

### Q18. Một SQL query production chậm. Quy trình debug của bạn là gì?

Trình tự tốt:
1. xác định query
2. EXPLAIN / EXPLAIN ANALYZE
3. rows estimated vs actual
4. scan type
5. join strategy
6. index
7. sort/temp work
8. query rewrite/schema nếu cần

### Q19. Tại sao database có index nhưng optimizer vẫn chọn sequential scan?

Nói về selectivity, table size, statistics, cost và số lượng row cần đọc.

### Q20. Composite index `(shop_id, created_at)` dùng tốt cho query nào?

Follow-up: leftmost prefix.

### Q21. Connection pool quá lớn gây vấn đề gì?

Nhiều connection không đồng nghĩa throughput cao hơn; DB có giới hạn CPU/memory/I/O và context switching.

### Q22. TimescaleDB hypertable giải quyết vấn đề gì?

Nói về time-series partitioning/chunking và quản lý dữ liệu theo thời gian.

### Q23. Compression phù hợp với dữ liệu nào?

Thường phù hợp dữ liệu lịch sử ít update hơn dữ liệu nóng đang thay đổi liên tục.

---

## 6. MongoDB

### Q24. MongoDB index khác SQL index về mặt tư duy thiết kế thế nào?

Trọng tâm vẫn là query pattern, cardinality/selectivity và write cost.

### Q25. MongoDB change stream là gì?

Follow-up quan trọng:
> Consumer mất kết nối một khoảng thời gian thì làm sao tránh mất event?

Nói về resume token/checkpoint và recovery strategy.

### Q26. Khi nào MongoDB phù hợp hơn PostgreSQL?

Không trả lời kiểu Mongo nhanh hơn SQL. So sánh data model, relationship, transaction/query requirements và schema evolution.

### Q27. MongoDB document lớn dần theo thời gian có rủi ro gì?

Nói về document growth, read/write amplification và document size limit.

---

## 7. Event / Analytics Pipeline — BigQuery

### Q28. Hãy thiết kế pipeline: event -> operational DB -> BigQuery.

Interviewer có thể hỏi:
- validate schema ở đâu?
- duplicate event xử lý thế nào?
- batching ở đâu?
- retry ra sao?
- BigQuery unavailable thì sao?

### Q29. Tại sao không ghi mọi request trực tiếp vào BigQuery?

Nói về coupling, latency, batching, failure isolation và operational workload vs analytics workload.

### Q30. Event schema thay đổi thì consumer cũ xử lý thế nào?

Có thể nói về backward compatibility, versioning, optional fields và schema validation.

### Q31. Làm thế nào đảm bảo event không bị mất khi pipeline gặp lỗi?

Nói về durable queue/storage, checkpoint, retry, DLQ và observability.

---

## 8. Shopify / OAuth / Security

### Q32. OAuth flow hoạt động như thế nào?

Nắm được authorization, callback, state validation và token handling.

### Q33. HMAC dùng để làm gì trong Shopify/webhook/API integration?

Interviewer có thể hỏi cách verify signature và tại sao phải dùng raw body trong một số webhook verification flow.

### Q34. Tại sao access token migration là thay đổi rủi ro?

Nói về backward compatibility, token expiry/refresh, rollout, fallback và monitoring.

### Q35. JWT và HMAC khác nhau thế nào?

Không chỉ định nghĩa; nói khi nào dùng signed token có claims và khi nào dùng request/message authentication.

### Q36. Làm thế nào chống webhook replay attack?

Có thể nói về signature verification, timestamp/window nếu protocol hỗ trợ và idempotency/event ID.

---

## 9. AWS / Cloud

### Q37. S3 và CloudFront kết hợp thế nào để deliver file?

Follow-up:
- private file thì sao?
- signed URL/cookie?
- cache invalidation?

### Q38. CloudWatch nên monitor những metrics nào cho backend worker?

Ví dụ:
- queue depth
- oldest message age
- processing latency
- error/retry rate
- DLQ count
- CPU/memory

### Q39. DynamoDB partition key chọn sai gây chuyện gì?

Nói về access pattern và hot partition.

### Q40. QLDB/immutable transaction log giải quyết bài toán gì?

Tập trung vào audit/history/tamper-evident use case; tránh khẳng định quá mức ngoài kinh nghiệm thực tế.

---

## 10. Production / Reliability

### Q41. Production incident xảy ra, 5 phút đầu bạn làm gì?

Câu trả lời tốt:
1. đánh giá impact
2. kiểm tra recent changes
3. metrics/logs/traces
4. mitigate trước nếu cần
5. rollback/disable feature
6. root cause sau khi hệ thống ổn định

### Q42. Retry có thể làm incident tệ hơn như thế nào?

Retry storm có thể tăng tải lên dependency đang lỗi. Dùng backoff, jitter, circuit breaking/rate limiting khi phù hợp.

### Q43. Health check nên kiểm tra gì?

Phân biệt liveness và readiness; tránh biến health endpoint thành một chuỗi dependency checks đắt đỏ.

### Q44. Zero-downtime migration DB làm thế nào?

Expand/contract pattern:
- add backward-compatible schema
- deploy code hỗ trợ cả hai
- migrate/backfill
- switch reads/writes
- remove old schema sau

---

## 11. Project Deep Dive — câu rất dễ gặp

### Q45. Project khó nhất bạn từng làm là gì?

Chuẩn bị một project có architecture + incident/trade-off rõ ràng.

### Q46. Một technical decision bạn từng đưa ra nhưng sau đó nhận ra chưa tối ưu?

Senior interviewer đánh giá khả năng nhìn nhận trade-off và học từ quyết định.

### Q47. Một production bug khó nhất bạn từng debug?

Chuẩn bị câu chuyện theo:
Symptom -> hypothesis -> investigation -> root cause -> fix -> prevention.

### Q48. Bạn từng cải thiện performance ở đâu? Đo bằng gì?

Đừng chỉ nói "thêm index". Nên có before/after metric nếu nhớ chính xác; nếu không nhớ thì không bịa số.

### Q49. Bạn từng thiết kế recovery mechanism nào?

Các chủ đề CV phù hợp để chuẩn bị: queue retry/DLQ, change-stream gap recovery, token migration rollback và event pipeline fallback.

### Q50. Nếu được thiết kế lại một hệ thống trong CV, bạn sẽ thay đổi gì?

Đây là câu rất tốt để thể hiện seniority: nói giới hạn của thiết kế cũ, constraint lúc đó và thiết kế mới nếu scale/requirements thay đổi.

---

# Top 15 cần thuộc trước

Nếu thời gian ít, ưu tiên:

1. Event loop và Node.js concurrency
2. SQS delivery semantics + idempotency
3. Retry/backoff/DLQ
4. Redis distributed lock failure cases
5. Index + composite index + selectivity
6. EXPLAIN ANALYZE
7. Transaction/isolation/deadlock
8. MongoDB indexing
9. Change stream recovery
10. TimescaleDB hypertable/compression
11. Event pipeline -> BigQuery
12. OAuth/HMAC/JWT
13. Feature-flagged migration
14. Production incident handling
15. Một project deep dive thật chắc

# Quy tắc quan trọng

Vì đây là câu hỏi sinh từ CV, hãy chỉ nhận là mình trực tiếp thiết kế/triển khai những phần thực sự đã làm. Với phần chỉ tham gia hoặc support, nói rõ phạm vi đóng góp. Senior interview thường đào rất nhanh qua những câu trả lời phóng đại.