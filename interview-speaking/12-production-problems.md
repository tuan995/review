# 12 — Production Problems & Debugging

Mục tiêu: khi interviewer hỏi **“Em từng gặp sự cố gì?”**, trả lời theo cách cho thấy quá trình điều tra chứ không chỉ nói tên lỗi và solution.

---

# 1. Framework trả lời incident

Dùng flow:

> **Biểu hiện → Ảnh hưởng → Em kiểm tra gì → Nguyên nhân → Cách sửa → Kiểm tra lại → Cách tránh tái diễn.**

## 🧾 Thuật ngữ

### **Symptom / Biểu hiện** *(thứ mình nhìn thấy bên ngoài)*

Ví dụ Prisma connection timeout.

### **Impact / Ảnh hưởng** *(người dùng hoặc hệ thống bị ảnh hưởng thế nào)*

Ví dụ job không đồng bộ đủ shop.

### **Root cause / Nguyên nhân gốc** *(nguyên nhân tạo ra biểu hiện, không chỉ tên error)*

Ví dụ `Promise.all` start quá nhiều query làm connection pool cạn.

### **Verification** *(kiểm tra sau fix xem vấn đề thật sự được giải quyết chưa)*

### **Prevention** *(biện pháp giảm khả năng lỗi tương tự quay lại)*

---

# 2. Case 1 — Prisma Connection Pool Timeout

## 💬 Bài nói 60–90 giây

> Trong một scheduled job em gặp Prisma timeout khi lấy database connection. Ban đầu error nhìn giống vấn đề database, nhưng em kiểm tra flow của job trước.
>
> Job đang dùng `Promise.all` để xử lý nhiều shop cùng lúc. Mỗi shop lại tạo nhiều query, nên số query active tăng rất nhanh. Connection pool có giới hạn; khi tất cả connection đang bận, query mới phải chờ và cuối cùng timeout.
>
> Em chuyển sang giới hạn số shop được xử lý đồng thời thay vì start toàn bộ danh sách. Sau đó em theo dõi lại error rate, processing time và database usage để chắc timeout giảm mà job vẫn đạt tốc độ chấp nhận được.
>
> Bài học của em là không chỉ tăng connection pool ngay. Trước tiên cần kiểm soát workload mà application đang tạo ra.

## 🎯 Interviewer hỏi tiếp

### Làm sao biết root cause là concurrency chứ không phải query chậm?

> Em nhìn timing của lỗi, flow của scheduled job, số task được start cùng lúc và connection usage. Nếu cần em xem query latency/execution plan để tách hai khả năng. Em không chỉ dựa vào một error message.

### Nếu query thực sự chậm thì sao?

> Khi đó concurrency limit chỉ giảm áp lực chứ chưa giải quyết nguyên nhân query. Em tiếp tục xem execution plan, index, data size và query pattern.

---

# 3. Case 2 — Nginx 413 / Upload Timeout

## 💬 Bài nói

> Khi upload file lớn, em từng gặp 413 hoặc timeout. Em tách request path thành client → Nginx → Node → storage để xác định layer nào đang từ chối hoặc chờ quá lâu.
>
> Với 413 em kiểm tra request body limit như `client_max_body_size`. Với timeout em xem loại timeout ở từng layer.
>
> Tuy nhiên nếu mục tiêu cuối chỉ là lưu file lên S3 thì thay vì tăng limit mãi, em ưu tiên direct upload bằng presigned URL để file không phải đi qua Node server.

### **Layer** *(một tầng trong request flow)*

### **Bottleneck** *(thành phần giới hạn tốc độ/khả năng của toàn flow)*

⚠️ **Dễ bị bắt bẻ:**

> “413 là do Nginx.”

✅ **Cách nói an toàn:**

> Nginx là một khả năng phổ biến nếu request đi qua Nginx, nhưng em kiểm tra layer thực tế vì application/proxy khác cũng có thể có body limit.

---

# 4. Case 3 — Cron chạy nhiều lần

## 💬 Bài nói

> Khi application chạy nhiều process hoặc container, nếu mỗi instance đều register cùng cron thì cùng một scheduled job có thể được trigger nhiều lần.
>
> Nếu job có side effect và không chịu được duplicate thì có thể tạo dữ liệu lặp hoặc gọi external API nhiều lần.
>
> Em có thể giải quyết bằng scheduler riêng, distributed coordination hoặc unique job trong queue tùy architecture. Đồng thời em vẫn cố thiết kế job idempotent để duplicate không làm sai dữ liệu.

### **Side effect** *(operation thay đổi trạng thái, ví dụ tạo record hoặc gọi payment API)*

### **Distributed coordination** *(cơ chế nhiều process/server thống nhất ai được thực hiện một việc)*

---

# 5. Case 4 — External API Timeout / 429

## 💬 Bài nói

> Khi external API fail em phân loại lỗi trước. Timeout, 429 hoặc một số 5xx có thể là lỗi tạm thời nên có retry policy. Validation hoặc authentication sai thì retry cùng request thường không giúp.
>
> Với 429 em tôn trọng metadata của provider nếu có và giảm request pressure bằng limiter, queue hoặc cache tùy flow.

### **Retry policy** *(quy tắc lỗi nào retry, chờ bao lâu, tối đa bao nhiêu lần)*

### **Request pressure** *(mức tải application đang tạo lên dependency)*

---

# 6. Case 5 — Memory tăng dần

## 💬 Bài nói

> Nếu Node process dùng memory tăng dần, em không restart rồi kết luận đã fix. Em xem traffic có tăng tương ứng không, heap usage có giảm sau GC không và object nào đang bị giữ lại.
>
> Những nguyên nhân em nghĩ tới gồm cache không giới hạn, global Map/array, listener không cleanup, timer hoặc closure giữ object lớn. Nếu cần em dùng heap snapshot để so sánh retained objects.

### **Heap** *(vùng memory runtime dùng để lưu nhiều JavaScript object)*

### **Heap snapshot** *(ảnh chụp trạng thái object trong heap để phân tích object nào đang chiếm/giữ memory)*

### **Memory leak** *(memory không được giải phóng dù dữ liệu đó không còn cần cho business)*

⚠️ Memory cao không tự động là memory leak.

---

# 7. Cách điều tra production issue

## Bước 1 — Xác định thời điểm

- bắt đầu lúc nào;
- có deploy/config change gần đó không;
- tất cả user hay một nhóm;
- xảy ra liên tục hay theo cron/traffic spike.

### **Timeline** *(trình tự thời gian của các sự kiện quanh incident)*

## Bước 2 — Thu thập evidence

- logs;
- metrics;
- error rate;
- latency;
- CPU/memory;
- DB connection/query;
- external API status.

### **Evidence** *(dữ liệu quan sát được dùng để kiểm chứng giả thuyết)*

## Bước 3 — Thu hẹp layer

Ví dụ:

```text
Client
  ↓
Nginx
  ↓
Node API
  ↓
Database / Redis / External API
```

Xác định request tới đâu thì bắt đầu lỗi.

## Bước 4 — Đưa ra hypothesis

### **Hypothesis** *(giả thuyết có thể kiểm chứng về nguyên nhân)*

Ví dụ:

> “Có thể job mới deploy làm connection pool cạn.”

Sau đó tìm metrics/log để chứng minh hoặc loại bỏ.

---

# 8. Logs nên có gì?

## 💬 Cách nói

> Em muốn log đủ context để nối các bước của một request/job: request ID hoặc job ID, operation, duration, result/error category và business identifier cần thiết. Nhưng em tránh log secret/token hoặc sensitive data.

### **Correlation ID** *(ID chung giúp nối log của cùng một request/workflow qua nhiều component)*

### **Structured log** *(log theo field như JSON thay vì chỉ một chuỗi text khó query)*

Ví dụ:

```json
{
  "jobId": "sync_123",
  "shop": "abc.myshopify.com",
  "durationMs": 843,
  "status": "failed",
  "errorType": "db_connection_timeout"
}
```

---

# 9. Metrics vs Logs

### **Logs**

Tốt để xem chi tiết một event/request cụ thể.

### **Metrics**

Tốt để biết xu hướng toàn hệ thống theo thời gian.

📌 Ví dụ metric:

- request count;
- error rate;
- p95 latency;
- DB connections;
- queue depth.

### **p95 latency** *(95% request có latency nhỏ hơn hoặc bằng giá trị này; 5% chậm hơn)*

⚠️ Average latency có thể che những request rất chậm.

---

# 10. Alert

## **Alert** *(cảnh báo khi metric/log vượt điều kiện cần chú ý)*

Alert tốt nên liên quan tới impact, không phải mọi warning đều page người trực.

Ví dụ:

- error rate tăng mạnh;
- queue oldest job > 30 phút;
- DB connection saturation kéo dài.

### **Saturation** *(resource gần hoặc đã dùng hết capacity)*

---

# 11. Fix xong chưa phải kết thúc

Sau fix:

1. verify error giảm;
2. kiểm tra side effect;
3. thêm metric/alert nếu trước đó thiếu;
4. thêm test/load test khi phù hợp;
5. ghi lại runbook hoặc note cho team.

### **Runbook** *(hướng dẫn thực tế để điều tra/xử lý một loại incident)*

---

# 🎯 Interviewer hỏi tiếp

### Nếu chưa tìm được root cause thì nói sao?

> Em nói rõ các hypothesis và thứ tự kiểm chứng. Em không đoán một solution rồi khẳng định đó là nguyên nhân khi chưa có evidence.

### Rollback deploy hay debug trước?

> Nếu impact lớn và deploy mới có khả năng cao liên quan, rollback có thể là bước giảm impact nhanh. Sau khi ổn định mới điều tra root cause sâu hơn. Quyết định tùy mức độ rủi ro và khả năng rollback.

### Monitoring và observability khác nhau sao?

> Monitoring thường tập trung vào metric/cảnh báo đã biết trước. Observability rộng hơn: dùng logs, metrics, traces để hiểu cả những failure mode mình chưa dự đoán cụ thể.

### Trace là gì?

> Trace theo dõi một request đi qua nhiều service/span để biết thời gian nằm ở đâu. Với distributed system nó giúp tìm component nào chậm.

---

# ⚠️ Những câu dễ bị bắt bẻ

❌ “Em nhìn log và tìm ra lỗi.”  
✅ “Em xác định timeline, xem log/metrics rồi kiểm chứng từng giả thuyết để thu hẹp root cause.”

❌ “Fix bằng restart server.”  
✅ “Restart có thể giảm symptom tạm thời; em vẫn cần tìm nguyên nhân nếu memory/resource tiếp tục tăng.”

❌ “Tăng timeout để fix timeout.”  
✅ “Em xác định operation đang chậm ở đâu; tăng timeout chỉ đúng nếu thời gian xử lý dài là hợp lệ và resource vẫn chịu được.”

---

# 📌 Công thức kể incident

> Em thấy **[symptom]**, ảnh hưởng **[impact]**. Em kiểm tra **[evidence]** và thu hẹp xuống **[layer]**. Nguyên nhân là **[root cause]**. Em sửa bằng **[fix]**, sau đó verify bằng **[metric/test]** và thêm **[prevention]**.

# Cách nhớ

**Thấy lỗi → đo → xác định layer → hypothesis → evidence → root cause → fix → verify → prevent**