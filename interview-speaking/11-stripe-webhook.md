# 11 — Stripe Subscription & Webhook

Mục tiêu: kể payment flow theo **trạng thái và nguồn dữ liệu đáng tin cậy**, đồng thời giải thích được SetupIntent, PaymentIntent, webhook, signature và idempotency.

> Lưu ý: flow Stripe phụ thuộc implementation của từng dự án. Khi interview nên mô tả đúng flow mình đã làm, không khẳng định một flow là duy nhất cho mọi hệ thống.

---

# 1. Cách kể payment flow

## 💬 Bài nói 60–90 giây

> Khi tích hợp Stripe, em tách hai phần: interaction ở client và trạng thái payment/subscription mà backend tin cậy.
>
> Client dùng Stripe SDK để nhập hoặc xác nhận thông tin thanh toán. Backend chịu trách nhiệm tạo các Stripe resource cần thiết và lưu mapping giữa Stripe customer/subscription với user hoặc shop trong hệ thống.
>
> Nhưng em không chỉ dựa vào việc client báo “payment thành công”. Client có thể đóng browser, mất network hoặc request response bị gián đoạn trong khi Stripe vẫn tiếp tục xử lý phía server.
>
> Vì vậy backend nhận **webhook** từ Stripe, verify chữ ký của webhook rồi xử lý event để cập nhật trạng thái vào database. UI sau đó đọc trạng thái từ backend.

---

# 🧾 Thuật ngữ

### **Payment provider** *(hệ thống bên ngoài xử lý payment, ví dụ Stripe)*

### **Stripe Customer** *(resource đại diện customer trong Stripe)*

### **Subscription** *(resource mô tả gói thanh toán định kỳ)*

### **Mapping** *(liên kết ID bên Stripe với user/shop/entity trong database của mình)*

📌 Ví dụ:

```text
shop_123
  ↕
stripe_customer_id = cus_xxx
stripe_subscription_id = sub_xxx
```

---

# 2. SetupIntent là gì?

## **SetupIntent** *(flow để thu thập/xác nhận payment method dùng cho payment tương lai mà không nhất thiết charge ngay lúc đó)*

## 💬 Cách nói

> Em dùng SetupIntent khi mục tiêu là setup payment method để có thể dùng cho future payment. Nó khác với flow thu tiền ngay.

### **Payment method** *(cách thanh toán đã được Stripe biểu diễn/tokenize, ví dụ card)*

⚠️ Không nói “SetupIntent là để tạo card”. Stripe quản lý payment method/resource theo flow riêng; cách nói an toàn hơn là “setup payment method cho future use”.

---

# 3. PaymentIntent là gì?

## **PaymentIntent** *(resource/flow đại diện cho quá trình cố gắng thu một khoản payment)*

## 💬 Cách nói

> PaymentIntent tập trung vào việc thu tiền và theo dõi trạng thái của lần thanh toán đó. SetupIntent tập trung vào setup payment method cho tương lai. Subscription flow có thể liên quan tới cả hai tùy trạng thái và cấu hình.

⚠️ **Dễ bị bắt bẻ:**

> “Subscription luôn dùng SetupIntent trước rồi PaymentIntent sau.”

✅ **Cách nói an toàn:**

> Flow cụ thể phụ thuộc payment behavior của subscription và implementation. Em sẽ mô tả đúng resource mà project của em tạo ở từng bước.

---

# 4. Webhook là gì?

## **Webhook** *(HTTP request Stripe chủ động gửi tới backend khi một event xảy ra)*

📌 Ví dụ Stripe có thể báo:

- invoice paid;
- payment failed;
- subscription updated;
- subscription deleted.

## 💬 Bài nói

> Webhook giúp backend biết trạng thái phía Stripe thay đổi mà không cần client phải luôn online. Khi nhận webhook em verify signature trước, sau đó xác định event type và update application state.

---

# 5. Tại sao phải verify webhook signature?

## **Signature verification** *(kiểm tra request thực sự được ký bằng secret của webhook endpoint chứ không phải request giả mạo)*

Nếu endpoint chỉ nhận JSON rồi tin luôn:

```text
Attacker → POST /stripe-webhook
        → { status: "paid" }
```

thì rất nguy hiểm.

## 💬 Cách nói

> Backend dùng webhook secret để verify signature trên raw request theo cơ chế Stripe cung cấp. Nếu verify fail thì không xử lý event như một event hợp lệ.

⚠️ Với một số framework, cần giữ đúng raw body cho signature verification. Khi interview không cần đi quá sâu nếu project không gặp issue này.

---

# 6. Idempotency khi xử lý webhook

## **Idempotency** *(cùng một event bị xử lý lại nhưng không tạo kết quả sai hoặc nhân đôi)*

Webhook có thể được retry hoặc delivered lại.

📌 Ví dụ nguy hiểm:

```text
invoice.paid event
   ↓ lần 1
cộng 100 credits
   ↓ event được gửi lại
cộng thêm 100 credits ❌
```

## Cách xử lý

Có thể:

- lưu Stripe event ID đã xử lý;
- dùng unique business constraint;
- thiết kế update theo state hiện tại;
- hoặc kết hợp các cách trên.

### **Deduplication** *(phát hiện event/request bị lặp và không thực hiện side effect lần nữa)*

---

# 7. Tại sao không tin client?

## 📌 Case

1. Client confirm payment.
2. Stripe xử lý thành công.
3. Browser mất mạng trước khi client gọi backend.
4. Nếu backend chỉ chờ client báo thì trạng thái local có thể vẫn `pending`.

## 💬 Cách nói

> Client là nơi tương tác với user, nhưng trạng thái payment quan trọng em đồng bộ từ Stripe qua backend/webhook. UI có thể poll backend hoặc refresh trạng thái sau đó.

### **Polling** *(client hỏi backend định kỳ xem trạng thái đã đổi chưa)*

---

# 8. Webhook đến trước API response thì sao?

Trong distributed system, thứ tự giữa:

- client API response;
- webhook;
- background worker;

không nên được giả định tuyệt đối.

## 💬 Cách nói

> Em thiết kế state update để cùng một thay đổi đến từ webhook hoặc flow API không làm state quay ngược hoặc tạo duplicate. Nếu cần, em dùng event ID, current state hoặc timestamp/version phù hợp.

### **Out-of-order event** *(event đến không đúng thứ tự mình tưởng tượng theo business flow)*

---

# 9. State machine cho payment

## **State machine** *(định nghĩa entity có những trạng thái nào và transition nào hợp lệ)*

Ví dụ đơn giản:

```text
pending
  ↓
active
  ↓
past_due / canceled
```

## 💬 Cách nói

> Em thích lưu trạng thái rõ ràng thay vì chỉ một boolean `isPaid`. Subscription có nhiều trạng thái trung gian và failure case, nên state rõ giúp webhook handler dễ quyết định update hợp lệ hơn.

⚠️ Trạng thái thực tế nên map theo business của application và Stripe resource, không nhất thiết copy toàn bộ Stripe status vào một enum mà không hiểu nghĩa.

---

# 10. Handler webhook có nên làm việc nặng không?

## 💬 Cách nói

> Nếu business logic dài, em ưu tiên verify event, persist/enqueue thông tin cần thiết rồi trả response thành công sớm. Worker xử lý phần nặng phía sau.

Lợi ích:

- webhook endpoint phản hồi nhanh;
- retry/internal failure dễ quản lý;
- không giữ HTTP request quá lâu.

### **Persist** *(lưu lại dữ liệu cần thiết để không phụ thuộc vào memory của process)*

### **Enqueue** *(đưa task vào queue để worker xử lý sau)*

---

# 11. Webhook fail thì sao?

Có hai tầng:

1. Stripe gửi webhook tới endpoint thất bại → provider có thể retry theo policy.
2. Endpoint nhận thành công nhưng internal processing fail → phía mình cần queue/retry/failed state phù hợp.

## **Reconciliation** *(job định kỳ kiểm tra Stripe và database để sửa state bị lệch)*

Với payment quan trọng, reconciliation là safety net tốt nếu webhook bị miss hoặc processing fail.

---

# 12. Security

Các nguyên tắc nên nhớ:

- secret key chỉ ở server;
- verify webhook signature;
- không tin amount/status do client tự gửi;
- tránh log card/sensitive payment data;
- dùng idempotency cho operation có thể retry;
- authorize user trước khi tạo payment flow liên quan account của họ.

### **Sensitive data** *(dữ liệu thanh toán/bảo mật không nên ghi log tùy ý)*

---

# 🎯 Interviewer hỏi tiếp

### SetupIntent vs PaymentIntent?

> SetupIntent dùng để setup payment method cho future payments; PaymentIntent đại diện cho flow thu một payment. Subscription có thể dùng flow khác nhau tùy cấu hình.

### Webhook có đảm bảo chỉ gửi một lần không?

> Em không thiết kế theo giả định đó. Handler phải chịu được duplicate/retry.

### Nếu xử lý webhook xong nhưng DB transaction fail?

> Endpoint nên chỉ mark event xử lý thành công khi phần cần thiết đã được persist theo flow của mình. Nếu dùng queue thì cần thiết kế ACK/retry/idempotency để event không bị mất.

### Nếu webhook bị miss hoàn toàn?

> Với state quan trọng em có thể có reconciliation job hoặc manual recovery path để đọc lại trạng thái từ Stripe.

### Idempotency key của API và event dedup có giống nhau không?

> Cùng mục tiêu tránh side effect lặp, nhưng một cái thường dùng khi client gửi lại API operation, còn webhook dedup dùng event/business identifier để tránh xử lý event lặp.

---

# ⚠️ Những câu dễ bị bắt bẻ

❌ “Client payment success thì em active subscription.”  
✅ “Client giúp hoàn thành interaction; backend đồng bộ trạng thái đáng tin cậy từ Stripe/webhook trước khi quyết định application state.”

❌ “Webhook realtime.”  
✅ “Webhook thường cập nhật gần thời điểm event, nhưng có thể delay/retry nên em không coi timing là tuyệt đối.”

❌ “Webhook verify bằng secret key Stripe.”  
✅ “Webhook signature dùng webhook signing secret của endpoint; secret API key là khái niệm khác.”

---

# 📌 Flow dễ nhớ

```text
Client bắt đầu payment/setup
        ↓
Backend tạo Stripe resource
        ↓
Stripe xử lý
        ↓
Webhook về backend
        ↓
Verify signature
        ↓
Dedup / idempotent processing
        ↓
Update DB / enqueue worker
        ↓
Client đọc trạng thái từ backend
```

# Cách nhớ

**Client interaction ≠ source of truth → Stripe event → verify → idempotency → state update → reconciliation nếu cần**