# 07 — API Integration & Rate Limit

# Case: Shopify / Google / Third-party API

## Bài nói

> Khi tích hợp third-party API, một vấn đề em luôn tính tới là rate limit. Nếu mỗi request của user lại gọi trực tiếp external API hoặc background job chạy fan-out lớn thì rất dễ vượt quota.
>
> Em xử lý theo nhiều lớp. Thứ nhất là giảm request không cần thiết bằng local database/cache. Thứ hai là giới hạn concurrency hoặc request rate của worker. Thứ ba là khi provider trả 429, em đọc thông tin rate-limit/retry mà API cung cấp và retry có backoff thay vì retry ngay lập tức.
>
> Với job không cần realtime, em ưu tiên queue để spread workload theo thời gian. Đồng thời cần metrics để biết request rate, error rate và số lần bị throttle.

### Rate limit và concurrency limit khác gì?

> Concurrency limit giới hạn số operation đang in-flight cùng lúc. Rate limit giới hạn số operation trong một khoảng thời gian. Một hệ thống có thể cần cả hai.

### 429 xử lý sao?

> Tôn trọng `Retry-After` hoặc metadata của provider nếu có. Nếu không, dùng exponential backoff có jitter và bounded retry.

### Tại sao cần jitter?

> Nếu nhiều worker cùng retry đúng một thời điểm thì có thể tạo thundering herd. Jitter làm thời điểm retry phân tán hơn.

### Retry mọi lỗi?

> Không. 429/timeout/5xx thường có khả năng transient. 400 validation hoặc authentication failure thường cần sửa input/configuration thay vì retry mù.

### Cache có giải quyết rate limit hoàn toàn?

> Không. Cache giảm read calls nhưng data freshness và invalidation trở thành trade-off. Write/sync operation vẫn cần rate control.

---

# Thiết kế tổng quát

```text
Application
    |
    v
local DB / cache
    |
    v
queue / limiter
    |
    v
Third-party API
    |
429 / 5xx
    |
backoff + retry + metrics
```

## Cách nhớ

`Reduce calls → limit → queue → 429 → backoff+jitter → monitor`
