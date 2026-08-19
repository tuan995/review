# 12 — Production Problems & Debugging

# Framework trả lời incident

Khi được hỏi **“Em gặp khó khăn gì?”**, đừng bắt đầu bằng solution.

> Symptom → Impact → Evidence → Root cause → Fix → Verification → Prevention

---

# Case 1 — Prisma connection pool timeout

> Hệ thống bắt đầu xuất hiện Prisma connection timeout trong một scheduled job. Em kiểm tra flow và thấy job dùng `Promise.all` để xử lý nhiều shop, mỗi shop lại tạo nhiều query.
>
> Root cause là fan-out concurrency vượt khả năng connection pool, không phải database tự nhiên bị lỗi.
>
> Em giới hạn concurrency và theo dõi lại error/processing time. Nếu cần tối ưu thêm thì mới cân nhắc pool size và query performance dựa trên DB capacity.

### Prevention?

> Bounded concurrency, metrics pool/query latency, load test job và tránh fan-out không giới hạn.

---

# Case 2 — Nginx 413 / upload timeout

> Khi upload file lớn, request bị 413 hoặc timeout. Em tách vấn đề theo layer: client → Nginx → Node → storage.
>
> 413 cho thấy cần kiểm tra request body limit ở proxy/application. Timeout cần xem client body/proxy/read timeout và thời gian xử lý. Nhưng thay vì chỉ tăng mọi timeout, với upload S3 em ưu tiên presigned direct upload để bỏ application server khỏi data path.

### Bài học?

> Debug production cần biết request đi qua những layer nào. Fix configuration có thể giải quyết symptom, nhưng thay đổi architecture đôi khi loại bỏ bottleneck tốt hơn.

---

# Case 3 — Cron chạy nhiều lần

> Khi application có nhiều process/instance, mỗi instance có thể schedule cùng cron. Nếu job không idempotent thì duplicate execution gây duplicate data hoặc external calls.
>
> Em giải quyết bằng cách đảm bảo chỉ một scheduler thực thi hoặc dùng distributed coordination/queue, đồng thời job vẫn nên idempotent để chịu được duplicate execution.

---

# Case 4 — External API timeout/429

> Em phân loại error trước. Timeout/5xx/429 thường transient nên có retry policy. Validation/auth errors thường không retry. Với 429 em tôn trọng provider rate-limit metadata, backoff và giảm request pressure bằng queue/cache/concurrency control.

---

# Câu hỏi đào sâu

### Làm sao tìm root cause?

> Em bắt đầu từ timeline và evidence: logs, metrics, error rate, latency, DB/API status và thay đổi deploy gần thời điểm incident. Sau đó thu hẹp từng layer thay vì sửa ngẫu nhiên.

### Log những gì?

> Request/job ID, operation, duration, result/error category và identifiers cần để correlate. Với distributed flow nên có correlation ID/tracing. Tránh log secret/token/sensitive data.

### Fix xong làm gì?

> Verify bằng metrics/test, thêm alert nếu cần và nghĩ cách prevention: concurrency limit, timeout, retry policy, idempotency, runbook hoặc test case.

### Nếu chưa biết nguyên nhân?

> Em nói rõ hypothesis và cách kiểm chứng từng hypothesis. Trong incident, một kế hoạch điều tra có evidence tốt hơn đoán solution.

## Cách nhớ

`See symptom → measure → isolate layer → root cause → fix → verify → prevent`
