# 11 — Stripe Subscription & Webhook

> Lưu ý khi luyện: dùng flow đúng với implementation thực tế của dự án. Không khẳng định một Stripe flow là duy nhất cho mọi integration.

# 1. Cách kể payment flow

## Bài nói

> Khi tích hợp payment, em tách hai khái niệm: client interaction và trạng thái thanh toán đáng tin cậy ở backend.
>
> Client có thể bắt đầu flow thanh toán hoặc setup payment method thông qua Stripe SDK. Backend tạo các Stripe resources cần thiết và lưu mapping giữa Stripe customer/subscription với user/shop của hệ thống.
>
> Nhưng em không chỉ tin trạng thái client gửi lên để quyết định subscription đã active hay payment đã thành công. Backend nhận Stripe webhook, verify signature rồi xử lý event để đồng bộ state vào database.
>
> Lý do là client có thể đóng browser, mất network hoặc response bị gián đoạn trong khi Stripe vẫn hoàn thành operation phía server.

---

# 2. SetupIntent

> SetupIntent được dùng khi muốn thu thập và thiết lập payment method cho các payment tương lai mà không nhất thiết charge ngay tại thời điểm setup. Với subscription flow cụ thể, em sẽ giải thích resource nào được tạo và tại sao dựa trên payment behavior của dự án.

### SetupIntent vs PaymentIntent?

> PaymentIntent đại diện cho flow thu tiền; SetupIntent tập trung setup payment method cho future payments. Subscription có thể liên quan tới PaymentIntent/SetupIntent tùy trạng thái và configuration.

---

# 3. Webhook

## Bài nói

> Webhook endpoint trước tiên verify Stripe signature bằng webhook secret. Sau đó em xác định event type và xử lý idempotently.
>
> Em không muốn handler thực hiện quá nhiều việc synchronous nếu operation dài. Có thể persist/enqueue event rồi trả 2xx sớm, worker xử lý business logic phía sau.

### Tại sao phải idempotent?

> Webhook có thể được retry/deliver lại. Nếu cùng event tạo order hoặc cộng credit hai lần thì dữ liệu sai. Có thể lưu Stripe event ID hoặc dùng unique business constraint để deduplicate.

### Nếu webhook đến trước response API?

> Backend design không nên phụ thuộc tuyệt đối vào ordering giữa client response và webhook. State update cần idempotent và có rule chuyển state rõ ràng.

### Nếu webhook fail?

> Provider có thể retry. Phía mình cần log/metrics, bounded internal retry nếu enqueue processing và reconciliation mechanism cho những state quan trọng.

---

# 4. Client polling / sync status

> Sau khi client hoàn thành interaction, UI có thể query backend để lấy trạng thái subscription/payment đã được backend đồng bộ. Backend là nơi trả application state thay vì client tự tuyên bố payment thành công.

---

# 5. Security

- Verify webhook signature.
- Không tin amount/status từ client.
- Secret key chỉ ở server.
- Idempotency cho operation có thể retry.
- Log identifier cần thiết nhưng tránh log sensitive payment data.

## Cách nhớ

`Client starts → backend creates Stripe resource → Stripe processes → webhook → verify → idempotent sync → client reads backend state`
