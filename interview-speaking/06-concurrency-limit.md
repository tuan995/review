# 06 — Promise.all, DB Connection Pool & Concurrency Limit

> Đây là case rất tốt cho câu "Em từng gặp vấn đề production nào?". Mục tiêu là kể được nguyên nhân từ đầu, không phụ thuộc vào các từ như fan-out, pool hay throughput mà chưa giải thích.

# 1. Bối cảnh

## Bài nói hoàn chỉnh

> Trong một background job em cần xử lý dữ liệu của nhiều shop. Mỗi shop cần đọc hoặc cập nhật một số dữ liệu trong database.
>
> Ban đầu em muốn job chạy nhanh nên em dùng `Promise.all` cho toàn bộ danh sách shop. Ví dụ có 50 shop thì application bắt đầu xử lý cả 50 shop gần như cùng lúc thay vì chờ shop này xong mới đến shop khác.
>
> Cách này hoạt động khi số lượng ít. Nhưng khi số shop và số query của mỗi shop tăng, application phát ra quá nhiều database query trong cùng một khoảng thời gian.

# 2. Vấn đề xảy ra

> Application sử dụng database connection pool. Pool có thể hiểu là một nhóm connection có sẵn để application tái sử dụng khi query database.
>
> Ví dụ pool cho phép khoảng 20 connection nhưng 50 shop cùng chạy và mỗi shop cần nhiều query. Khi tất cả connection đang bận, query mới phải chờ connection được trả về pool. Nếu chờ quá thời gian cấu hình thì Prisma báo timeout khi lấy connection.
>
> Vì vậy vấn đề không phải `Promise.all` bị lỗi. Vấn đề là em tạo nhiều công việc đồng thời hơn capacity của resource phía dưới.

### Downstream resource là gì?

> Là hệ thống/resource mà code của mình gọi tới phía sau, ví dụ PostgreSQL, Redis, Shopify API hoặc một service khác. Application có thể tạo request rất nhanh nhưng downstream không có capacity vô hạn.

### Capacity là gì?

> Capacity là mức tải mà resource có thể xử lý ổn định trong điều kiện cụ thể. Ví dụ database chỉ có số connection hữu hạn và CPU/I/O hữu hạn.

---

# 3. Tại sao `Promise.all` tạo vấn đề?

> `Promise.all` nhận nhiều Promise và chờ chúng hoàn thành, nhưng nó không phải concurrency limiter. Nếu em tạo Promise cho toàn bộ 1.000 item, các operation bên trong có thể đều được start rất sớm.
>
> Với I/O, điều đó có thể biến thành burst: trong thời gian ngắn application gửi rất nhiều query/API request xuống hệ thống khác.

### Burst là gì?

> Burst là tải tăng mạnh trong một khoảng thời gian ngắn. Tổng số request trong cả phút có thể không quá lớn, nhưng nếu hàng trăm request dồn vào cùng một giây thì downstream vẫn có thể quá tải.

---

# 4. Giải pháp: Bounded Concurrency

## Bounded concurrency là gì?

> Bounded concurrency nghĩa là em vẫn xử lý nhiều task cùng lúc, nhưng đặt một giới hạn rõ ràng. Ví dụ tối đa 5 shop đang chạy. Khi một shop xong thì slot trống mới nhận shop tiếp theo.

```text
50 shops
   ↓
chỉ 5 task active
[1] [2] [3] [4] [5]
          ↓ một task xong
        lấy shop tiếp theo
```

> Em có thể implement bằng worker pool nhỏ, concurrency limiter như `p-limit`, hoặc queue có cấu hình worker concurrency.

```ts
const limit = pLimit(5);

await Promise.all(
  shops.map(shop => limit(() => processShop(shop)))
);
```

> Ở đây vẫn có `Promise.all`, nhưng `limit()` đảm bảo chỉ tối đa 5 `processShop` thực sự active tại một thời điểm.

---

# 5. Tại sao không xử lý tuần tự?

> Nếu chỉ `for ... await` từng shop một thì an toàn hơn về tải nhưng có thể lãng phí thời gian trong lúc một shop đang chờ I/O. Bounded concurrency là điểm cân bằng: vẫn tận dụng thời gian chờ I/O nhưng không cho số task tăng vô hạn.

### Throughput là gì?

> Throughput là lượng công việc hoàn thành trong một đơn vị thời gian. Ví dụ job xử lý được bao nhiêu shop/phút. Mục tiêu không phải lúc nào cũng là concurrency cao nhất mà là throughput tốt trong khi latency/error/resource usage vẫn ổn định.

### Latency là gì?

> Latency là thời gian một operation mất từ lúc bắt đầu tới lúc có kết quả. Khi database quá tải, tăng concurrency có thể làm latency tăng vì query phải chờ nhau.

---

# 6. Tại sao không chỉ tăng connection pool?

> Tăng pool có thể đúng nếu database còn capacity. Nhưng connection không miễn phí: mỗi connection dùng resource phía database.
>
> Nếu nguyên nhân là application có thể tạo workload không giới hạn thì tăng từ 20 lên 100 chỉ làm ngưỡng lỗi xuất hiện muộn hơn. Khi số shop tiếp tục tăng, vấn đề có thể quay lại.
>
> Vì vậy em ưu tiên kiểm soát concurrency trước, sau đó mới tune pool dựa trên database capacity và metrics.

---

# 7. Batch và concurrency limit khác nhau thế nào?

### Batch

```text
[1 2 3 4 5] → chờ cả 5 xong
[6 7 8 9 10] → bắt đầu
```

> Nếu task 1–4 xong nhanh nhưng task 5 rất chậm thì 4 slot bị bỏ trống trong lúc chờ task 5.

### Worker pool / limiter

```text
1 xong → lấy 6 ngay
2 xong → lấy 7 ngay
...
```

> Vì vậy worker pool giữ tối đa N task active nhưng không cần đợi cả nhóm cùng xong. Resource utilization thường tốt hơn.

### Resource utilization là gì?

> Là mức độ mình sử dụng resource có sẵn. Ví dụ có thể an toàn chạy 5 query đồng thời thì worker pool cố giữ các slot đó có việc, thay vì để 4 slot rảnh chỉ vì một task trong batch đang chậm.

---

# 8. Chọn concurrency = 5 hay 10 thế nào?

> Em không chọn theo cảm tính. Em xem connection pool size, một shop thường tạo bao nhiêu query, query mất bao lâu, database CPU/connection usage, error rate và nếu có external API thì cả rate limit của API.
>
> Sau đó benchmark/load test với các mức concurrency khác nhau. Nếu tăng từ 5 lên 10 giúp throughput nhưng database vẫn ổn thì có thể tăng. Nếu latency/error tăng mạnh thì cần giảm hoặc tối ưu query.

### Benchmark là gì?

> Là đo performance dưới một workload xác định để so sánh các cấu hình/cách làm, thay vì đoán rằng cách nào nhanh hơn.

---

# 9. Nếu process chết giữa job?

> Promise đang nằm trong memory sẽ mất khi process chết. Nếu đây là job quan trọng, em cần lưu trạng thái hoặc dùng persistent queue để task có thể được chạy lại.

### Persistent queue là gì?

> Là queue lưu job ở một storage không biến mất chỉ vì worker process restart, ví dụ Redis-backed queue tùy công nghệ. Worker lấy job, xử lý và acknowledge/mark complete theo cơ chế của queue.

### Retry có nguy hiểm không?

> Có nếu operation không idempotent. Ví dụ retry `create payment` mù quáng có thể tạo hai payment. Vì vậy retry thường phải đi cùng idempotency hoặc cơ chế kiểm tra trạng thái.

---

# 10. STAR version để trả lời nhanh

**Situation**

> Em có background job xử lý dữ liệu của nhiều shop.

**Task**

> Em cần giảm thời gian job nhưng không làm database quá tải.

**Action**

> Ban đầu dùng `Promise.all` toàn bộ danh sách nên số query concurrent tăng mạnh và connection pool timeout. Em xác định nguyên nhân là unbounded concurrency, sau đó giới hạn số shop active cùng lúc và theo dõi DB metrics để chọn mức phù hợp.

**Result**

> Job có thể lâu hơn một chút so với việc cố start mọi thứ cùng lúc, nhưng database ổn định hơn, giảm connection timeout và workload dễ dự đoán hơn.

# 11. Interviewer đào tiếp

### Fan-out là gì nếu em dùng từ này?

> Fan-out nghĩa là từ một operation ban đầu mình mở rộng thành rất nhiều operation con cùng lúc. Ví dụ một cron chạy một lần nhưng lập tức start xử lý 50 shop, mỗi shop lại tạo nhiều query.

### Unbounded concurrency là gì?

> Là số task đồng thời không có giới hạn rõ ràng và thường tăng theo kích thước input. 50 item thành 50 task, 5.000 item có thể thành hàng nghìn task nếu code không giới hạn.

### Backpressure có liên quan không?

> Có cùng tư tưởng kiểm soát tốc độ. Khi phía tạo công việc nhanh hơn phía xử lý, mình cần cơ chế làm phía tạo chậm lại hoặc giới hạn lượng work đang in-flight để resource không tăng mất kiểm soát.

## Cách nhớ

`50 shop → Promise.all start quá nhiều → query dồn xuống DB → pool hết connection → query chờ/timeout → giới hạn 5–10 task → đo metrics → ổn định hơn`
