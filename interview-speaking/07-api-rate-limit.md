# 07 — API Integration & Rate Limit

Mục tiêu: giải thích rate limit bằng **flow request thực tế**, không chỉ nói “retry + backoff”.

---

# 1. Rate limit là gì?

## **Rate limit** *(giới hạn số request được phép trong một khoảng thời gian hoặc theo một cơ chế quota cụ thể)*

Ví dụ provider có thể giới hạn số request/giây, request/phút hoặc dùng cost/token bucket tùy API.

## 💬 Bài nói

> Khi tích hợp Shopify, Google hoặc API bên thứ ba, em luôn tính tới việc provider không cho mình gọi vô hạn. Nếu mỗi request user đều gọi trực tiếp external API, hoặc một background job cùng lúc xử lý hàng trăm item, mình có thể vượt giới hạn và nhận lỗi 429 hoặc bị throttle.
>
> Em xử lý theo nhiều lớp. Đầu tiên là giảm request không cần thiết bằng local database/cache. Sau đó em giới hạn số request hoặc số task đang chạy. Nếu provider trả rate-limit response thì em đọc metadata/`Retry-After` nếu có và retry sau một khoảng phù hợp thay vì gọi lại ngay.
>
> Với job không cần realtime, queue cũng giúp trải workload theo thời gian thay vì dồn request trong một burst.

---

# 🧾 Thuật ngữ

### **Quota** *(hạn mức provider cấp cho client/application)*

### **Throttle** *(provider chủ động làm chậm hoặc từ chối request khi vượt giới hạn)*

### **429 Too Many Requests** *(HTTP status thường dùng khi client gửi quá nhiều request)*

### **Retry-After** *(header/metadata cho biết nên chờ bao lâu trước khi thử lại, nếu provider cung cấp)*

### **Burst** *(nhiều request dồn vào một khoảng thời gian rất ngắn)*

---

# 2. Rate limit vs Concurrency limit

## **Concurrency limit** *(giới hạn số operation đang active cùng lúc)*

## **Rate limit** *(giới hạn số operation trong một khoảng thời gian)*

📌 Ví dụ:

- Concurrency = 5: tối đa 5 request đang chờ response cùng lúc.
- Rate = 10 req/s: trong một giây không gửi quá 10 request.

Một hệ thống có thể cần cả hai.

⚠️ **Dễ bị nhầm:** concurrency = 5 không đảm bảo rate <= 10 req/s nếu mỗi request hoàn thành cực nhanh.

---

# 3. Retry thế nào?

## 💬 Bài nói

> Em không retry ngay lập tức và cũng không retry vô hạn. Nếu provider trả `Retry-After` thì ưu tiên tôn trọng thông tin đó. Nếu không có, em có thể dùng backoff — tức là mỗi lần thất bại thì chờ trước khi thử lại.

### **Exponential backoff** *(thời gian chờ tăng dần sau mỗi lần retry)*

Ví dụ:

```text
Lần 1 fail → chờ ~1s
Lần 2 fail → chờ ~2s
Lần 3 fail → chờ ~4s
```

### **Jitter** *(thêm một khoảng ngẫu nhiên nhỏ vào thời gian chờ)*

Nếu 100 worker cùng nhận 429 và tất cả cùng chờ chính xác 2 giây, sau 2 giây chúng lại cùng gửi request → tạo burst mới.

Jitter làm thời điểm retry phân tán hơn.

### **Bounded retry** *(retry có số lần/thời gian giới hạn)*

Không nên retry vô hạn vì có thể làm queue tăng mãi hoặc che giấu lỗi permanent.

---

# 4. Lỗi nào nên retry?

## Có khả năng retry

- timeout;
- một số network error;
- 429;
- nhiều trường hợp 5xx.

Đây thường là **transient errors** *(lỗi có khả năng chỉ xảy ra tạm thời)*.

## Thường không retry mù

- 400 vì payload invalid;
- authentication/authorization sai;
- resource không hợp lệ theo business rule.

Đây có thể là **permanent error** *(gửi lại cùng request cũng không tự hết)*.

⚠️ Không nên học status code thành bảng tuyệt đối; luôn xem semantics của API cụ thể.

---

# 5. Tại sao dùng local database/cache?

## 📌 Ví dụ

Frontend cần hiển thị product title 1.000 lần/ngày.

Nếu mỗi lần đều gọi Shopify thì:

- tăng latency;
- dùng quota;
- phụ thuộc availability của Shopify.

Nếu dữ liệu phù hợp để lưu local, application có thể đọc database/cache và chỉ sync khi cần.

### **Cache** *(bản dữ liệu tạm giúp giảm số lần đọc nguồn chậm hơn)*

⚠️ Cache không giải quyết rate limit hoàn toàn. Write/sync vẫn cần limiter và cache tạo thêm bài toán freshness/invalidation.

---

# 6. Queue giúp gì?

## **Queue** *(hàng đợi lưu các task để worker lấy xử lý dần)*

Ví dụ cần sync 10.000 products nhưng không cần hoàn thành trong một HTTP request.

```text
10.000 jobs
    ↓
Queue
    ↓
Worker lấy từng job với rate/concurrency control
    ↓
Shopify API
```

Queue giúp:

- tránh burst;
- retry độc lập;
- kiểm soát số worker;
- recover tốt hơn nếu process restart.

---

# 7. Idempotency khi retry

## **Idempotency** *(cùng một operation chạy lại nhưng không tạo kết quả sai hoặc nhân đôi)*

📌 Nếu retry `GET`, thường ít nguy hiểm hơn. Nhưng retry `POST create payment` có thể tạo hai payment nếu request đầu đã thành công nhưng response bị mất.

## 💬 Cách nói

> Với operation có side effect, em chỉ retry khi có cơ chế nhận biết request lặp, ví dụ idempotency key hoặc business key/unique constraint tùy hệ thống.

### **Side effect** *(operation làm thay đổi trạng thái bên ngoài, ví dụ tạo order/charge tiền)*

---

# 8. Monitoring rate limit

Chỉ retry là chưa đủ. Em muốn biết:

- request rate;
- số response 429;
- retry count;
- latency;
- error rate;
- queue depth nếu dùng queue.

### **Metrics** *(số liệu đo theo thời gian để biết hệ thống đang hoạt động thế nào)*

---

# 🎯 Interviewer hỏi tiếp

### Tại sao không chỉ sleep 1 giây giữa mọi request?

> Cách đó đơn giản nhưng có thể quá chậm hoặc vẫn không đúng policy của provider. Em ưu tiên limiter theo rate-limit model thực tế và metadata provider trả về.

### Nếu nhiều shop có rate limit riêng thì sao?

> Em có thể cần limiter theo từng shop/token thay vì một global limiter, tùy API policy. Quan trọng là giới hạn phải map đúng với resource/provider đang giới hạn.

### Nếu queue backlog tăng liên tục?

> Có thể worker xử lý chậm hơn tốc độ job được tạo. Em xem throughput, rate limit, lỗi retry, số worker và downstream capacity trước khi chỉ tăng worker.

### Circuit breaker có liên quan không?

> Có thể dùng khi external service lỗi liên tục để tạm ngừng gửi request thay vì tiếp tục gây tải. Nhưng em chỉ thêm khi hệ thống thực sự cần; không phải mọi integration đều cần circuit breaker.

### **Circuit breaker** *(tạm dừng gọi dependency khi lỗi liên tục, rồi thử lại sau)*

---

# ⚠️ Những câu dễ bị bắt bẻ

❌ “429 thì retry 3 lần.”  
✅ “Em đọc policy/Retry-After của provider và retry có giới hạn; số lần phụ thuộc use case.”

❌ “Cache sẽ tránh rate limit.”  
✅ “Cache giảm một phần read calls; write/sync và cache miss vẫn cần kiểm soát.”

❌ “Concurrency limit và rate limit giống nhau.”  
✅ “Một cái giới hạn số task active, một cái giới hạn số request theo thời gian.”

---

# 📌 Cách nhớ

**Giảm request → giới hạn concurrency/rate → queue nếu cần → gặp 429 → chờ/backoff+jitter → retry có giới hạn → đo metrics**