# 11 — Stripe Subscription & Webhook

> Lưu ý khi luyện: chỉ kể đúng flow mình thực sự đã làm. Stripe có nhiều cách cấu hình subscription/payment nên không nói một flow là đúng cho mọi dự án.

# 1. Cách kể payment flow

## Bài nói

> Khi tích hợp Stripe, em tách hai phần: thao tác user thực hiện trên client và trạng thái thanh toán mà backend tin tưởng để lưu vào hệ thống.
>
> Client có thể nhập hoặc xác nhận payment method thông qua Stripe SDK. Backend tạo các resource cần thiết bên Stripe và lưu ID để biết Stripe customer/subscription nào thuộc user hoặc shop nào trong hệ thống của mình.
>
> Nhưng em không chỉ dựa vào việc client báo “payment thành công”. Client có thể đóng tab, mất mạng hoặc không nhận được response dù phía Stripe đã xử lý xong.
>
> Vì vậy backend nhận webhook từ Stripe. Khi có event quan trọng, backend kiểm tra webhook thật sự đến từ Stripe rồi cập nhật trạng thái tương ứng trong database. UI sau đó đọc trạng thái từ backend của mình.

### Mapping là gì?

> Ở đây mapping chỉ là việc lưu mối liên hệ giữa ID bên Stripe và record bên hệ thống mình. Ví dụ `stripeCustomerId` nào thuộc shop nào, hoặc `subscriptionId` nào thuộc user nào.

---

# 2. SetupIntent và PaymentIntent

### SetupIntent là gì?

> SetupIntent được dùng khi mình muốn thiết lập payment method để có thể dùng cho payment sau này mà không nhất thiết phải charge ngay tại bước setup.

### PaymentIntent là gì?

> PaymentIntent đại diện cho quá trình thu một khoản tiền và trạng thái của quá trình đó, ví dụ còn cần user xác nhận hay đã thành công/thất bại.

### SetupIntent khác PaymentIntent thế nào?

> Em nhớ đơn giản: SetupIntent tập trung vào **chuẩn bị payment method cho tương lai**, còn PaymentIntent tập trung vào **một flow thu tiền cụ thể**.
>
> Với subscription, Stripe có thể dùng các resource này khác nhau tùy cách cấu hình và trạng thái subscription, nên khi phỏng vấn em mô tả đúng flow dự án thay vì nói subscription lúc nào cũng dùng một loại cố định.

---

# 3. Webhook

## Webhook là gì trong Stripe?

> Webhook là endpoint backend của mình để Stripe chủ động gửi sự kiện, ví dụ payment thành công, payment thất bại hoặc subscription thay đổi trạng thái.

## Bài nói

> Khi Stripe gọi webhook, bước đầu tiên của em là verify signature để kiểm tra request được ký bằng webhook secret mà hệ thống mình cấu hình với Stripe.
>
> Sau đó em đọc event type và xử lý dữ liệu tương ứng. Với event quan trọng, em lưu hoặc kiểm tra event ID để nếu Stripe gửi lại cùng event thì hệ thống không cập nhật sai hoặc tạo duplicate record.
>
> Nếu xử lý business logic khá dài, em không muốn giữ webhook request mở quá lâu. Em có thể lưu/enqueue event rồi trả response thành công sớm, sau đó worker xử lý phía sau.

### Verify signature là gì?

> Stripe gửi kèm chữ ký được tạo từ webhook payload và secret. Backend dùng secret của mình để kiểm tra chữ ký đó. Mục tiêu là tránh việc ai đó tự gọi endpoint webhook và giả làm Stripe.

### Event type là gì?

> Là loại sự kiện Stripe đang thông báo, ví dụ một payment đã thành công hay subscription đã thay đổi. Mỗi loại event dẫn tới business logic khác nhau.

### Idempotent là gì trong webhook?

> Nghĩa là cùng một event bị gửi lại nhiều lần nhưng kết quả cuối không bị nhân đôi. Ví dụ cùng một payment-success event tới hai lần nhưng hệ thống vẫn chỉ ghi nhận một payment tương ứng.

### Tại sao Stripe có thể gửi lại webhook?

> Vì phía Stripe không thể chắc backend của mình đã xử lý thành công chỉ dựa vào việc gửi request một lần. Nếu request timeout hoặc backend trả lỗi, provider có thể retry delivery.

### Nếu webhook đến trước response API thì sao?

> Em không thiết kế hệ thống phụ thuộc tuyệt đối vào thứ tự giữa client response và webhook. Hai luồng này có thể đến khác thứ tự do network.
>
> Backend nên cập nhật state theo rule rõ ràng và xử lý lại cùng event an toàn. UI chỉ cần query backend để lấy trạng thái hiện tại.

---

# 4. Queue / xử lý webhook phía sau

### Enqueue là gì?

> Là đưa một task/event vào queue để worker xử lý sau, thay vì thực hiện toàn bộ công việc ngay trong HTTP request webhook.

### Tại sao trả 2xx sớm?

> Nếu endpoint giữ request quá lâu, provider có thể coi delivery thất bại và retry. Nếu flow cho phép, em xác nhận đã nhận event sau khi verify/lưu an toàn rồi để worker xử lý phần dài phía sau.

### Nếu worker xử lý event fail thì sao?

> Em retry có giới hạn. Nếu vẫn fail thì giữ lại trạng thái lỗi để điều tra. Với những trạng thái payment quan trọng, em có thể có job kiểm tra lại với Stripe để tránh dữ liệu local bị lệch lâu.

### “Reconciliation” trong payment là gì?

> Là định kỳ kiểm tra lại trạng thái từ Stripe và so với database của mình, sau đó sửa những record bị lệch. Ví dụ webhook bị miss nhưng Stripe đã chuyển subscription sang trạng thái khác.

---

# 5. Client lấy trạng thái thế nào?

> Sau khi user thao tác xong, frontend có thể gọi backend để lấy trạng thái subscription/payment. Backend trả trạng thái đã lưu hoặc đã đồng bộ từ Stripe.
>
> Em tránh để client tự gửi một field kiểu `paymentSuccess: true` rồi backend tin đó là nguồn xác nhận cuối cùng.

### Polling là gì?

> Là client gọi backend lặp lại theo một khoảng thời gian để hỏi trạng thái đã thay đổi chưa. Ví dụ mỗi vài giây hỏi subscription đã active chưa. Nếu dùng polling em cần giới hạn tần suất hợp lý.

---

# 6. Security — nói đơn giản

- Stripe secret key chỉ nằm ở server, không gửi xuống browser.
- Webhook phải verify signature.
- Không tin amount/status do client tự gửi nếu backend có thể lấy từ Stripe hoặc dữ liệu đã xác thực.
- Operation có thể retry phải tránh tạo duplicate.
- Log đủ ID để debug nhưng tránh log card data hoặc secret.

### Sensitive data là gì?

> Là dữ liệu cần bảo vệ, ví dụ secret key, token hoặc thông tin thanh toán nhạy cảm. Khi debug em ưu tiên log identifier như payment ID/event ID thay vì log toàn bộ payload nhạy cảm.

---

# Bài nói 60–90 giây

> Với Stripe subscription, em không coi client là nơi quyết định cuối cùng payment đã thành công hay chưa. Client dùng Stripe SDK để thực hiện phần cần tương tác với user, còn backend tạo/lưu các Stripe resource cần thiết.
>
> Khi Stripe có thay đổi quan trọng, backend nhận webhook, verify signature rồi cập nhật trạng thái trong database. Webhook có thể gửi lại nên handler cần xử lý cùng event nhiều lần mà không tạo duplicate.
>
> Nếu xử lý dài em lưu event hoặc đưa vào queue rồi để worker xử lý. Frontend sau đó đọc trạng thái từ backend. Cách này giúp hệ thống vẫn đồng bộ được kể cả khi user đóng browser hoặc client không nhận được response cuối.

## Cách nhớ

`Client thao tác → backend tạo/lưu Stripe ID → Stripe xử lý → webhook báo backend → verify → tránh duplicate → DB lưu state → client đọc lại từ backend`
