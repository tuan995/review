# 02 — Project Experience: Shopify & Backend Integration

## Cách giới thiệu dự án

> Trong một số dự án gần đây em phát triển backend cho các ứng dụng liên quan tới Shopify. Backend chịu trách nhiệm nhận và đồng bộ dữ liệu từ Shopify, lưu dữ liệu cần thiết vào database, xử lý webhook/background jobs và cung cấp API cho frontend.
>
> Vì phụ thuộc vào external API nên bài toán không chỉ là CRUD. Em phải xử lý rate limit, dữ liệu thay đổi ở Shopify, retry khi request fail và đảm bảo dữ liệu local không lệch quá lâu so với source of truth.

---

# Case 1 — Tại sao lưu database thay vì gọi Shopify liên tục?

## Bài nói

> Có những dữ liệu em có thể lấy trực tiếp từ Shopify API, nhưng nếu frontend hoặc các internal service gọi Shopify cho mọi request thì hệ thống phụ thuộc rất mạnh vào latency và availability của Shopify, đồng thời dễ chạm rate limit.
>
> Vì vậy với dữ liệu cần đọc thường xuyên, em lưu một representation cần thiết trong database của hệ thống. Request đọc thông thường sẽ lấy từ database, còn Shopify vẫn là source of truth và dữ liệu local được cập nhật thông qua webhook hoặc synchronization job.
>
> Trade-off của cách này là xuất hiện bài toán data consistency. Database local có thể stale trong một khoảng thời gian, nên tùy loại dữ liệu em quyết định dữ liệu nào cần webhook gần real-time, dữ liệu nào eventual consistency là chấp nhận được.

### Tại sao không gọi Shopify trực tiếp?

> Vì latency cao hơn, phụ thuộc external service và khó kiểm soát rate limit khi traffic tăng. Local database cũng giúp query/filter/report thuận tiện hơn.

### Nhược điểm của lưu local database?

> Phải giải quyết synchronization, duplicate event, missing webhook và stale data.

### Shopify hay database là source of truth?

> Với dữ liệu Shopify quản lý thì Shopify vẫn là source of truth. Database của em là local representation phục vụ application.

---

# Case 2 — Inventory thay đổi liên tục nhưng job chỉ chạy mỗi ngày

## Bài nói

> Nếu inventory thay đổi liên tục mà hệ thống chỉ sync một lần mỗi ngày thì dữ liệu có thể stale tới gần 24 giờ. Với inventory, khoảng trễ như vậy thường quá lớn.
>
> Em sẽ không giải quyết chỉ bằng cách tăng cron từ một ngày xuống vài phút, vì polling liên tục vừa tốn API quota vừa vẫn có delay.
>
> Hướng tốt hơn là webhook-driven synchronization: khi inventory thay đổi, Shopify gửi webhook và hệ thống cập nhật dữ liệu local. Cron vẫn giữ lại nhưng đóng vai trò reconciliation job để định kỳ kiểm tra và sửa những event bị miss.

### Tại sao vẫn cần cron nếu đã có webhook?

> Webhook giúp gần real-time nhưng không nên giả định delivery luôn hoàn hảo. Reconciliation job là safety net để hệ thống eventual consistent trở lại.

### Nếu webhook gửi trùng thì sao?

> Handler cần idempotent. Có thể lưu event identifier hoặc thiết kế update sao cho xử lý cùng event nhiều lần không làm sai state.

### Nếu event đến sai thứ tự?

> Nếu domain nhạy với ordering, em so sánh version/timestamp hoặc fetch lại state hiện tại từ source of truth thay vì áp dụng mù event cũ.

---

# Case 3 — External API failure

## Bài nói

> Khi tích hợp external API em luôn coi failure là trạng thái bình thường cần thiết kế trước. Request có thể timeout, trả 429 hoặc 5xx.
>
> Với lỗi transient em retry có giới hạn và dùng exponential backoff. Với rate limit em tôn trọng retry information từ provider nếu có. Với lỗi permanent như invalid input hoặc authentication thì retry liên tục không có ý nghĩa, cần log/alert hoặc đưa job vào trạng thái failed để xử lý.

### Retry bao nhiêu lần?

> Không có một con số đúng cho mọi API. Em dựa vào SLA, loại operation và policy của provider. Quan trọng là bounded retry, backoff và observability.

### Retry POST có nguy hiểm không?

> Có, vì request đầu tiên có thể đã thành công nhưng response bị mất. Với operation tạo resource/payment nên dùng idempotency key hoặc cơ chế deduplication.

## Cách nhớ chương

`External API → local DB → webhook → reconciliation → idempotency → retry`
