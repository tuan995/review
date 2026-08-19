# 06 — Promise.all, DB Connection Pool & Concurrency Limit

Đây là một case rất tốt để trả lời câu **“Em từng gặp vấn đề production nào?”**.

# Case

## Bài nói hoàn chỉnh

> Trong một background job em cần xử lý dữ liệu của nhiều shop. Ban đầu em dùng `Promise.all` trên toàn bộ danh sách để các shop được xử lý đồng thời và job hoàn thành nhanh hơn.
>
> Khi số lượng shop tăng lên khoảng vài chục, mỗi shop lại thực hiện nhiều database query. Kết quả là application tạo ra số lượng query concurrent lớn hơn khả năng phục vụ của database connection pool và Prisma bắt đầu báo timeout khi lấy connection.
>
> Ban đầu có thể nghĩ tới tăng connection pool, nhưng em thấy đó chỉ tăng giới hạn chứ không kiểm soát nguyên nhân là unbounded concurrency.
>
> Em thay đổi flow thành bounded concurrency: tại một thời điểm chỉ xử lý một số shop nhất định, ví dụ 5 hoặc 10 tùy benchmark. Khi một task hoàn thành mới lấy task tiếp theo.
>
> Tổng thời gian job có thể dài hơn `Promise.all` không giới hạn, nhưng throughput ổn định hơn, database không bị burst quá lớn và hệ thống predictable hơn.

---

# Interviewer đào sâu

### `Promise.all` có sai không?

> Không. Với số task nhỏ hoặc resource chịu được tải thì rất hữu ích. Vấn đề là dùng nó trên collection lớn mà không nghĩ tới downstream capacity.

### Tại sao không tăng pool từ 20 lên 100?

> Có thể tăng nếu database có capacity và benchmark chứng minh cần thiết, nhưng connection cũng tiêu tốn resource ở DB. Nếu workload vẫn unbounded thì tăng pool chỉ dời điểm failure.

### Concurrency và parallelism?

> Concurrency là nhiều task cùng in-flight/progress trong một khoảng thời gian. Parallelism là thực sự execute đồng thời, ví dụ trên nhiều CPU cores. Với Node API, nhiều I/O operation có thể concurrent dù JavaScript callback chủ yếu chạy trên một main thread.

### Implement limit thế nào?

> Có thể chia batch, viết worker pool nhỏ hoặc dùng thư viện limiter như `p-limit`. Với distributed workload lớn hơn em sẽ dùng queue với worker concurrency rõ ràng.

```ts
const limit = pLimit(5);
await Promise.all(shops.map(shop => limit(() => processShop(shop))));
```

### Batch 5 và concurrency 5 có giống nhau?

> Không hoàn toàn. Batch đợi cả nhóm hoàn thành rồi mới chạy nhóm tiếp. Worker pool/concurrency limiter có thể lấy task mới ngay khi một slot trống, nên resource utilization thường tốt hơn.

### Làm sao chọn 5 hay 10?

> Dựa vào pool size, số query mỗi task, DB capacity, API limits và benchmark/metrics. Không chọn một con số cố định theo cảm tính.

### Nếu process chết giữa job?

> Nếu job quan trọng em không chỉ dựa vào Promise trong memory. Em dùng persistent queue/job state, retry và idempotency để task có thể tiếp tục an toàn.

---

# STAR version

**Situation:** Job xử lý nhiều shop.

**Task:** Đồng bộ nhanh nhưng không làm ảnh hưởng database.

**Action:** Xác định fan-out từ `Promise.all`, kiểm tra connection pool, chuyển sang bounded concurrency.

**Result:** Giảm connection timeout và làm workload ổn định hơn, đổi lại job có thể chạy lâu hơn một chút.

## Cách nhớ

`50 shops → Promise.all → many queries → pool timeout → limit concurrency → stable DB`
