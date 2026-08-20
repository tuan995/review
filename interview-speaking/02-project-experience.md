# 02 — Project Experience: Shopify & Backend Integration

Mục tiêu của chương này là kể project theo **flow hệ thống**, không chỉ liệt kê công nghệ.

---

# 💬 Cách giới thiệu dự án

> Trong một số dự án gần đây em phát triển backend cho các ứng dụng liên quan tới Shopify. Backend chịu trách nhiệm nhận dữ liệu từ Shopify, lưu những dữ liệu cần thiết vào database, xử lý webhook và background job, sau đó cung cấp API cho frontend hoặc các service khác.
>
> Vì hệ thống phụ thuộc vào Shopify và các external API nên bài toán không chỉ là CRUD. Em phải quan tâm tới giới hạn API, request có thể timeout, webhook có thể bị gửi lại, dữ liệu local có thể chậm hơn dữ liệu trên Shopify và job có thể chạy lỗi giữa chừng.
>
> Vì vậy khi thiết kế flow em thường nghĩ cả trường hợp thành công lẫn failure case ngay từ đầu.

---

# 🧾 Thuật ngữ cơ bản

### **External API** *(API do hệ thống khác cung cấp)*

Ví dụ Shopify Admin API, Stripe API hoặc Google API.

### **Source of truth** *(nơi được coi là dữ liệu chính thức)*

Nếu Shopify là nơi quản lý inventory chính thức thì khi database local và Shopify khác nhau, Shopify thường là nơi mình dùng để kiểm tra lại.

### **Stale data** *(dữ liệu local chưa kịp cập nhật theo dữ liệu mới nhất)*

Ví dụ inventory trên Shopify đã là 3 nhưng database local vẫn đang giữ 5 vì webhook chưa được xử lý.

### **Eventual consistency** *(dữ liệu không nhất thiết giống nhau ngay lập tức nhưng hệ thống được thiết kế để sau một khoảng thời gian sẽ đồng bộ lại)*

Khi phỏng vấn nên giải thích hiện tượng trước rồi mới dùng thuật ngữ này.

---

# Case 1 — Tại sao lưu database thay vì gọi Shopify liên tục?

## 💬 Bài nói 60–90 giây

> Có những dữ liệu em có thể lấy trực tiếp từ Shopify API. Nhưng nếu mỗi lần frontend cần dữ liệu mà backend lại gọi Shopify thì response phụ thuộc vào network và thời gian phản hồi của Shopify. Khi traffic tăng, số request tới Shopify cũng tăng và có thể chạm rate limit.
>
> Vì vậy với những dữ liệu application cần đọc thường xuyên, em lưu phần dữ liệu cần thiết trong database của hệ thống. Request đọc thông thường sẽ lấy từ database local nên nhanh và chủ động hơn.
>
> Shopify vẫn là nơi quản lý dữ liệu chính đối với các thông tin thuộc Shopify. Database local giống một bản dữ liệu phục vụ application. Khi Shopify thay đổi, em cập nhật bản local bằng webhook hoặc job đồng bộ.
>
> Đổi lại, khi lưu dữ liệu ở hai nơi thì có khả năng hai bên lệch nhau trong một khoảng thời gian. Vì vậy mình phải thiết kế cách cập nhật và cách kiểm tra lại dữ liệu.

## 🧠 Hiểu sâu

Lưu local database không có nghĩa “database của mình đúng hơn Shopify”. Lợi ích chính là:

- giảm số request external API;
- giảm phụ thuộc vào latency của hệ thống bên ngoài;
- dễ query/filter/report theo nhu cầu riêng;
- hệ thống vẫn có thể đọc một số dữ liệu khi external service gặp vấn đề tạm thời.

Đổi lại, application phải xử lý bài toán **synchronization** *(đồng bộ dữ liệu giữa hai hệ thống)*.

## 🎯 Interviewer hỏi tiếp

### Tại sao không dùng cache thôi?

> Cache phù hợp nếu dữ liệu chỉ cần lưu tạm để giảm số lần đọc. Nhưng nếu em cần query, relation, history hoặc dùng dữ liệu đó cho business logic thì database local thường phù hợp hơn. Có thể dùng cả database và cache nếu cần.

### Shopify hay database là source of truth?

> Với dữ liệu do Shopify quản lý như product/inventory thì Shopify thường là nguồn chính. Database local là bản phục vụ application và phải có cơ chế cập nhật lại khi bị lệch.

### Làm sao biết dữ liệu local bị lệch?

> Em có thể dùng webhook để cập nhật gần thời điểm thay đổi và thêm một job định kỳ đọc lại dữ liệu quan trọng từ Shopify để kiểm tra. Nếu phát hiện khác nhau thì cập nhật lại database local.

---

# Case 2 — Inventory thay đổi liên tục nhưng job chỉ chạy mỗi ngày

## 💬 Bài nói

> Nếu inventory thay đổi nhiều lần trong ngày nhưng hệ thống chỉ đồng bộ một lần mỗi ngày thì database local có thể sai trong nhiều giờ. Với tồn kho, độ trễ như vậy thường không phù hợp.
>
> Cách đầu tiên có thể nghĩ tới là chạy cron thường xuyên hơn, ví dụ vài phút một lần. Nhưng nếu cứ polling toàn bộ dữ liệu liên tục thì vừa tốn API request vừa vẫn luôn có một khoảng trễ.
>
> Em ưu tiên dùng webhook cho những thay đổi cần cập nhật nhanh. Khi Shopify báo inventory thay đổi, backend nhận webhook và cập nhật dữ liệu local.
>
> Tuy nhiên em vẫn giữ một job định kỳ để kiểm tra lại vì webhook có thể bị miss, xử lý lỗi hoặc có vấn đề trong deployment. Job này thường gọi là **reconciliation job** — tức là job kiểm tra hai bên và sửa dữ liệu bị lệch.

## 📌 Flow dễ nhớ

```text
Shopify thay đổi inventory
        ↓
Webhook gửi tới backend
        ↓
Backend update database local
        ↓
Cron định kỳ kiểm tra lại
        ↓
Nếu lệch → sửa lại từ Shopify
```

## 🧾 Thuật ngữ

### **Polling** *(chủ động hỏi API lặp lại theo thời gian để xem dữ liệu có thay đổi không)*

Ví dụ cứ 5 phút gọi Shopify để hỏi inventory hiện tại.

### **Webhook-driven** *(để bên có thay đổi chủ động báo cho mình thay vì mình hỏi liên tục)*

### **Reconciliation** *(kiểm tra lại và sửa chênh lệch)*

📌 Ví dụ job ban đêm so sánh inventory local với Shopify rồi cập nhật những product bị lệch.

## 🎯 Interviewer hỏi tiếp

### Tại sao vẫn cần cron nếu đã có webhook?

> Webhook giúp cập nhật nhanh nhưng em không coi delivery là tuyệt đối không bao giờ lỗi. Cron đóng vai trò safety net, tức là lớp kiểm tra lại để dữ liệu có thể trở về đúng nếu một webhook bị bỏ lỡ.

### Nếu webhook gửi trùng thì sao?

> Handler nên chịu được cùng một event chạy lại mà không tạo kết quả sai. Tính chất đó gọi là **idempotency**. Ví dụ cùng webhook update inventory = 3 chạy hai lần vẫn chỉ cho kết quả inventory = 3 chứ không trừ thêm hai lần.

### Nếu webhook đến sai thứ tự thì sao?

> Với dữ liệu nhạy với thứ tự, em không áp dụng mù event cũ. Có thể so sánh timestamp/version nếu nguồn cung cấp, hoặc đọc lại state hiện tại từ Shopify trước khi quyết định cập nhật.

---

# Case 3 — External API bị lỗi

## 💬 Bài nói

> Khi tích hợp API bên ngoài em coi lỗi là tình huống bình thường cần thiết kế trước. Request có thể timeout, provider có thể trả 429 vì vượt giới hạn, hoặc 5xx khi service của họ gặp vấn đề.
>
> Em không retry tất cả lỗi giống nhau. Nếu lỗi có khả năng chỉ xảy ra tạm thời, ví dụ timeout hoặc 5xx, em có thể retry có giới hạn. Giữa các lần retry em chờ một khoảng thay vì gọi lại ngay để tránh tạo thêm tải.
>
> Với lỗi input sai hoặc authentication sai thì retry lặp lại thường không giúp gì; lúc đó cần log, alert hoặc đưa job sang trạng thái failed để xử lý.

## 🧾 Thuật ngữ

### **Transient error** *(lỗi có khả năng chỉ xảy ra tạm thời)*

Ví dụ network timeout hoặc service trả 503.

### **Permanent error** *(lỗi sẽ không tự hết nếu gửi lại cùng request)*

Ví dụ access token sai hoặc payload validation sai.

### **Retry** *(thử lại operation sau khi lỗi)*

### **Exponential backoff** *(mỗi lần retry sau chờ lâu hơn lần trước)*

Ví dụ chờ 1 giây → 2 giây → 4 giây.

### **Jitter** *(thêm một khoảng ngẫu nhiên nhỏ vào thời gian chờ)*

Mục đích là tránh hàng trăm worker cùng retry đúng một giây.

---

# Case 4 — Retry POST có thể tạo dữ liệu trùng

## 📌 Ví dụ

Application gửi request tạo payment. Stripe thực tế đã tạo payment nhưng response bị mất do network timeout. Application tưởng thất bại và gửi lại request.

Nếu không có cơ chế bảo vệ, có thể tạo hai payment.

## 💬 Cách nói

> Với operation tạo dữ liệu hoặc charge tiền, em không retry mù. Em cần một cách để provider hoặc hệ thống của mình nhận ra hai request thực chất là cùng một nghiệp vụ. Cách này thường gọi là **idempotency key** hoặc deduplication tùy API.

### **Deduplication** *(phát hiện và bỏ qua dữ liệu/request bị lặp)*

---

# ⚠️ Dễ bị bắt bẻ

## “Webhook đảm bảo realtime.”

Không nên nói tuyệt đối.

✅ **Cách nói an toàn:**

> Webhook giúp cập nhật gần thời điểm thay đổi hơn polling theo lịch, nhưng vẫn cần xử lý retry, duplicate và trường hợp event bị miss.

## “Database local luôn consistent với Shopify.”

✅ **Cách nói an toàn:**

> Database local có thể chậm hơn trong một khoảng ngắn, nhưng em có webhook và job kiểm tra lại để giảm thời gian dữ liệu bị lệch.

## “Mọi lỗi 5xx em retry 3 lần.”

Con số cứng dễ bị hỏi “tại sao là 3?”.

✅ **Cách nói an toàn:**

> Em retry có giới hạn dựa trên loại operation, policy của provider và thời gian chấp nhận được của nghiệp vụ.

---

# 📌 Cách nhớ chương

**External API → local DB → webhook → dữ liệu có thể lệch → reconciliation → retry → idempotency**

Nếu nhớ được chuỗi này, bạn có thể tự kể lại toàn bộ chapter bằng lời của mình.