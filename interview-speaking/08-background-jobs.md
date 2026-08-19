# 08 — Background Job, Cron & Queue

# 1. Khi nào dùng background job?

## Bài nói

> Em đưa công việc ra background khi task không cần hoàn thành trong request-response hiện tại, chạy lâu hoặc cần retry độc lập. Ví dụ synchronization nhiều shop, gửi email, xử lý file hoặc reconciliation.
>
> Request có thể tạo job rồi trả response sớm. Worker xử lý phía sau, nhờ đó latency của API không bị phụ thuộc vào task dài.

---

# 2. Cron vs Queue

> Cron phù hợp công việc theo lịch, ví dụ reconciliation mỗi ngày. Queue phù hợp event/task cần xử lý reliable, retry và scale worker.
>
> Hai thứ có thể kết hợp: cron tạo jobs vào queue thay vì cron process toàn bộ workload trong một process.

### Tại sao cách đó tốt hơn?

> Queue cho phép concurrency control, retry, visibility và phân phối workload tốt hơn. Nếu cron process trực tiếp hàng nghìn item thì restart giữa chừng khó biết item nào đã hoàn thành.

---

# 3. Duplicate cron khi chạy nhiều instance

## Bài nói

> Khi application scale nhiều PM2/container instances, nếu mỗi instance đều register cùng cron thì một job có thể chạy nhiều lần.
>
> Với job cần singleton execution, em có thể tách scheduler riêng, dùng distributed lock hoặc để scheduler enqueue một unique job vào queue. Chỉ check instance number có thể dùng trong deployment đơn giản nhưng không phải giải pháp distributed tổng quát.

### Distributed lock có rủi ro gì?

> Cần TTL, ownership/token và xử lý process chết. Lock implementation sai có thể gây duplicate hoặc lock không bao giờ release.

---

# 4. Retry & Idempotency

> Background job phải giả định có thể chạy lại. Nếu worker xử lý xong nhưng crash trước khi acknowledge thì queue có thể redeliver. Vì vậy operation quan trọng nên idempotent hoặc có deduplication/business key.

### Dead-letter queue?

> Job fail quá số lần retry được chuyển sang nơi riêng để inspect/reprocess thay vì retry vô hạn.

---

# 5. Monitoring

> Em muốn biết queue depth, processing latency, success/failure rate, retry count và age của oldest job. Chỉ log “job started” là chưa đủ để biết hệ thống có backlog hay không.

## Cách nhớ

`Schedule → enqueue → workers → concurrency → retry → idempotency → DLQ → metrics`
