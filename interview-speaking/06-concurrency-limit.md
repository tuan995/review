# 06 — Promise.all, DB Connection Pool & Concurrency Limit

Đây là một case rất tốt cho câu **“Em từng gặp vấn đề production nào?”** vì nó nối được Node.js, database và system design.

---

# 1. Bối cảnh

## 💬 Bài nói 60–90 giây

> Trong một background job em cần xử lý dữ liệu của nhiều shop. Ban đầu em dùng `Promise.all` trên toàn bộ danh sách để job chạy nhanh hơn.
>
> Khi số shop còn ít thì không có vấn đề. Nhưng khi số lượng tăng lên vài chục shop, mỗi shop lại thực hiện nhiều database query, nên trong một khoảng thời gian application tạo ra rất nhiều query cùng lúc.
>
> Application sử dụng **connection pool** *(một nhóm database connection được tái sử dụng)*. Khi tất cả connection đều bận, query mới phải chờ. Nếu chờ quá lâu thì Prisma báo timeout khi lấy connection.
>
> Em xác định nguyên nhân không phải `Promise.all` “bị lỗi”, mà là số task đồng thời vượt quá khả năng database phục vụ ổn định. Em chuyển sang giới hạn số shop được xử lý cùng lúc, ví dụ 5 hoặc 10 tùy metrics và benchmark.
>
> Cách này làm tổng thời gian job có thể dài hơn một chút, đổi lại database ổn định hơn và giảm connection timeout.

---

# 2. `Promise.all` thực sự làm gì?

## 💬 Cách nói

> `Promise.all` nhận một danh sách Promise và trả về một Promise hoàn thành khi tất cả thành công, hoặc reject khi một Promise reject. Nó không có cơ chế tự giới hạn số operation mà mình đã start.

📌 Ví dụ:

```ts
await Promise.all(shops.map(shop => processShop(shop)));
```

Nếu `shops` có 1.000 phần tử, code có thể start rất nhiều `processShop()` trong thời gian rất ngắn.

### **Unbounded concurrency** *(số task đồng thời không có giới hạn rõ ràng và tăng theo input)*

50 item có thể thành 50 task; 5.000 item có thể thành hàng nghìn task.

### **Fan-out** *(một operation ban đầu mở rộng thành nhiều operation con)*

Ví dụ một cron start 50 shop, mỗi shop lại tạo 10 query → có thể tạo hàng trăm query.

⚠️ **Dễ bị bắt bẻ:**

> “`Promise.all` chạy mọi thứ parallel.”

✅ **Cách nói an toàn:**

> `Promise.all` giúp start/chờ nhiều asynchronous operation cùng nhau. Việc chúng thực sự chạy parallel hay chỉ concurrent phụ thuộc loại operation và runtime phía dưới.

---

# 3. Connection Pool

## **Connection pool** *(nhóm kết nối database được giữ sẵn để tái sử dụng)*

Application không nhất thiết mở connection mới cho từng query rồi đóng ngay. Pool giữ một số connection để tái sử dụng.

📌 Ví dụ:

```text
Pool có 20 connection
      ↓
20 query đang dùng hết
      ↓
query thứ 21 phải chờ
      ↓
chờ quá lâu
      ↓
connection acquisition timeout
```

### **Connection acquisition timeout** *(query chờ lấy connection từ pool quá lâu rồi bị timeout)*

### Tại sao không tăng pool từ 20 lên 100?

> Có thể tăng nếu database thực sự còn capacity và benchmark cho thấy phù hợp. Nhưng connection cũng dùng resource phía database. Nếu application vẫn tạo workload không giới hạn thì tăng pool chỉ có thể dời điểm failure sang một mức cao hơn.

---

# 4. Bounded Concurrency

## **Bounded concurrency** *(vẫn chạy nhiều task cùng lúc nhưng có giới hạn tối đa)*

Ví dụ tối đa 5 shop active:

```text
[1] [2] [3] [4] [5]
          ↓ task 3 xong
[1] [2] [6] [4] [5]
```

## 📌 Ví dụ code

```ts
const limit = pLimit(5);

await Promise.all(
  shops.map(shop => limit(() => processShop(shop)))
);
```

Ở đây vẫn dùng `Promise.all`, nhưng limiter chỉ cho tối đa 5 `processShop` thực sự active.

### **Limiter** *(cơ chế giới hạn số operation được phép active)*

---

# 5. Tại sao không xử lý từng shop tuần tự?

## 💬 Cách nói

> Nếu chỉ xử lý một shop mỗi lần thì database rất an toàn nhưng có thể lãng phí thời gian khi shop đó đang chờ I/O. Bounded concurrency cho phép một số shop cùng chạy để tận dụng thời gian chờ, nhưng không để số task tăng vô hạn.

### **Throughput** *(số lượng công việc hoàn thành trong một đơn vị thời gian)*

Ví dụ 100 shop/phút.

### **Latency** *(thời gian một task mất từ lúc bắt đầu tới lúc hoàn thành)*

Concurrency tăng không có nghĩa latency luôn giảm. Khi database quá tải, query có thể phải chờ nhau lâu hơn và latency tăng.

---

# 6. Batch vs Worker Pool

## Batch

```text
[1 2 3 4 5] → đợi cả 5 xong
[6 7 8 9 10] → mới bắt đầu
```

Nếu task 5 rất chậm, 4 slot còn lại có thể bị rảnh.

## Worker pool / limiter

```text
1 xong → lấy 6 ngay
2 xong → lấy 7 ngay
```

### **Worker pool** *(một số worker/slot cố định liên tục lấy task tiếp theo khi rảnh)*

### **Resource utilization** *(mức độ sử dụng resource có sẵn)*

Worker pool thường giữ các slot hoạt động đều hơn batch cứng.

---

# 7. Chọn concurrency = 5 hay 10 như thế nào?

## 💬 Cách nói

> Em không chọn theo cảm tính. Em xem pool size, số query trung bình mỗi shop, query latency, database CPU/connection usage, error rate và cả rate limit của external API nếu job có gọi API.
>
> Sau đó em benchmark hoặc load test với các mức khác nhau. Nếu tăng từ 5 lên 10 giúp throughput tốt hơn mà latency/error/database usage vẫn ổn thì có thể tăng. Nếu error hoặc waiting tăng mạnh thì giảm lại hoặc tối ưu query.

### **Benchmark** *(đo performance với một workload xác định để so sánh)*

### **Load test** *(tạo tải có kiểm soát để xem hệ thống phản ứng khi số request/task tăng)*

---

# 8. Burst và downstream capacity

### **Burst** *(rất nhiều request/task dồn vào một khoảng thời gian ngắn)*

Tổng số request trong một phút có thể không lớn, nhưng nếu 500 request dồn vào cùng một giây thì database vẫn có thể quá tải.

### **Downstream** *(hệ thống phía sau mà application gọi tới)*

Ví dụ PostgreSQL, Redis, Shopify API.

### **Capacity** *(mức tải một resource có thể xử lý ổn định trong điều kiện cụ thể)*

⚠️ Không có một con số capacity vĩnh viễn; nó phụ thuộc hardware, query, workload và configuration.

---

# 9. Process chết giữa job thì sao?

Promise đang nằm trong memory sẽ mất khi process chết.

Nếu job quan trọng, em cần:

- lưu trạng thái tiến trình;
- hoặc dùng queue lưu job ở storage bền hơn;
- retry task chưa hoàn thành;
- đảm bảo task chạy lại không tạo dữ liệu sai.

### **Persistent queue** *(queue lưu job ở storage để worker restart vẫn còn job)*

### **Idempotency** *(task chạy lại không tạo kết quả sai hoặc nhân đôi)*

📌 Ví dụ retry sync product có thể update lại cùng record, nhưng không nên tạo duplicate product.

---

# 10. Backpressure liên quan gì?

## **Backpressure** *(làm phía tạo công việc chậm lại khi phía xử lý không theo kịp)*

Ý tưởng giống concurrency limit:

> Nếu producer tạo task nhanh hơn database xử lý, mình không tiếp tục đẩy vô hạn. Mình giới hạn lượng work đang active hoặc queue lại.

---

# 🎯 Interviewer hỏi tiếp

### `Promise.all` reject thì những task khác có dừng không?

> Không tự động. Aggregate Promise reject khi một Promise reject, nhưng những operation đã start không tự nhiên bị cancel. Muốn cancel cần cơ chế riêng tùy operation.

### Tăng pool có khi nào là đúng?

> Có. Nếu database còn resource và current pool quá nhỏ so với workload bình thường thì tăng có thể hợp lý. Em chỉ không dùng tăng pool như cách thay thế cho việc kiểm soát workload không giới hạn.

### Nếu job cần chạy nhanh hơn nữa?

> Em có thể tối ưu query, giảm số query mỗi shop, batch database operations, tăng worker concurrency trong mức DB chịu được hoặc scale worker. Nhưng em đo bottleneck trước khi tăng concurrency.

### Có nên dùng queue không?

> Nếu job lớn, cần retry, cần recover khi process restart hoặc cần chia workload cho nhiều worker thì queue hợp lý hơn một `Promise.all` nằm trong memory.

---

# ⚠️ Dễ bị bắt bẻ

❌ “Concurrency càng cao càng nhanh.”  
✅ “Tăng concurrency chỉ giúp tới khi resource phía dưới bắt đầu contention/queueing; sau điểm đó latency và error có thể tăng.”

❌ “Pool 20 thì concurrency nên bằng 20.”  
✅ “Không thể suy ra trực tiếp vì một task có thể dùng nhiều query, còn application có request khác cùng dùng pool.”

❌ “Batch 5 giống limiter 5.”  
✅ “Cùng giới hạn số task theo nhóm nhưng batch cứng phải chờ cả nhóm; limiter có thể lấy task mới ngay khi một slot rảnh.”

---

# 📌 STAR version

**Situation:** Background job xử lý nhiều shop.  
**Task:** Rút ngắn thời gian nhưng không làm database quá tải.  
**Action:** Phát hiện `Promise.all` start quá nhiều query → kiểm tra connection pool → giới hạn concurrency → theo dõi metrics.  
**Result:** Giảm connection timeout và làm workload ổn định hơn, đổi lại job có thể dài hơn một chút.

---

# Cách nhớ

**50 shop → Promise.all → query dồn xuống DB → pool hết connection → query chờ/timeout → giới hạn task → đo lại → ổn định hơn**