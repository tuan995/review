# 03 — Node.js & Event Loop

# 1. Giới thiệu Node.js

## Bài nói

> Node.js là JavaScript runtime chạy trên V8. Điểm em thấy quan trọng nhất khi làm backend với Node.js là mô hình event-driven và non-blocking I/O.
>
> JavaScript của application chủ yếu chạy trên một main thread. Nhưng nói Node.js single-threaded không có nghĩa toàn bộ runtime chỉ có một thread. Network I/O có thể dựa vào cơ chế async của OS, còn một số operation như filesystem, DNS lookup, crypto hoặc zlib có thể sử dụng libuv worker pool.
>
> Nhờ vậy một process có thể quản lý nhiều I/O operation mà không cần tạo một thread riêng cho mỗi request. Tuy nhiên đổi lại, nếu em chạy CPU-heavy JavaScript quá lâu trên main thread thì Event Loop bị block và các request khác phải chờ.

### Node.js phù hợp bài toán nào?

> I/O-bound workload như API server, realtime service hoặc hệ thống có nhiều network/database I/O.

### Khi nào Node.js không phù hợp?

> CPU-heavy workload nếu computation được chạy trực tiếp lâu trên main thread. Khi đó cần Worker Threads, background workers hoặc tách service tùy bài toán.

---

# 2. Event Loop

## Bài nói

> Khi Node chạy JavaScript, synchronous code được thực thi trên main thread. Nếu gặp asynchronous I/O, Node đăng ký operation với runtime/OS/libuv thay vì đứng chờ nó hoàn thành.
>
> Khi operation sẵn sàng, callback tương ứng sẽ được xử lý thông qua Event Loop khi main thread có thể chạy nó.
>
> Nếu cần nói sâu hơn, Event Loop có các phase liên quan tới timers, pending callbacks, poll, check và close callbacks. `setImmediate` gắn với check phase, còn timer được xử lý theo timer scheduling. Promise callbacks và `process.nextTick` có cơ chế queue ưu tiên riêng và được drain ở các điểm thích hợp giữa quá trình xử lý callbacks.

## Sơ đồ nhớ

```text
JS main thread
     |
     | async operation
     v
OS / libuv / worker pool
     |
     | completion/event
     v
Event Loop / queues
     |
     v
callback chạy trên JS thread
```

### Event Loop có những phase nào?

> Em thường nhớ các phase quan trọng là timers, pending callbacks, poll, check và close callbacks. Khi interview em tập trung giải thích ý nghĩa thay vì chỉ đọc tên phase.

### `setImmediate()` và `setTimeout(fn, 0)`?

> `setImmediate` được xử lý ở check phase. `setTimeout(0)` là timer với minimum delay chứ không có nghĩa chạy ngay. Không nên nói một cái luôn chạy trước cái còn lại trong mọi context.

### Promise chạy ở đâu?

> Callback của Promise là microtask. Sau khi current JavaScript/callback hoàn thành, microtasks có cơ hội được xử lý trước khi tiếp tục các callback thông thường của event loop.

### `process.nextTick()` thì sao?

> Node có nextTick queue và nó được ưu tiên trước Promise microtasks khi Node drain các queue này. Vì vậy nếu liên tục tạo `nextTick` callback thì có thể làm I/O bị starvation.

---

# 3. libuv & Thread Pool

## Bài nói

> Một điểm em từng hiểu nhầm khi mới học Node là cứ asynchronous thì sẽ tạo thread mới. Thực tế không phải vậy.
>
> Network I/O thường sử dụng non-blocking socket và polling mechanism của OS. libuv cũng có worker pool cho những operation không phù hợp với cơ chế đó, ví dụ nhiều filesystem APIs, `dns.lookup`, một số crypto và zlib operations.
>
> Vì vậy `async/await` hay Promise không phải cơ chế tạo thread. Chúng là abstraction để tổ chức asynchronous control flow.

### Default libuv thread pool size?

> Thông thường default là 4 và có thể điều chỉnh bằng `UV_THREADPOOL_SIZE`, nhưng tăng pool không phải lúc nào cũng làm hệ thống nhanh hơn; vẫn phải xét CPU, workload và contention.

---

# 4. CPU-bound

## Bài nói

> Nếu một request chạy một vòng lặp computation rất lớn thì dù function được khai báo `async`, đoạn computation synchronous đó vẫn chiếm main thread.
>
> Khi main thread bận, Event Loop không thể cho các callback/request khác chạy kịp thời. Đây là lý do Node xử lý I/O concurrency tốt nhưng cần cẩn thận với CPU-heavy work.

### Giải quyết thế nào?

> Nếu CPU work thực sự cần chạy trong cùng application em cân nhắc Worker Threads. Nếu task dài và không cần trả kết quả ngay, em thường thích đưa vào queue/background worker. Nếu workload độc lập và cần scale riêng thì có thể tách service.

---

# 5. async/await

## Bài nói

> `async/await` là syntax trên Promise giúp asynchronous code dễ đọc và quản lý error hơn. Khi `await` một Promise đang pending, function hiện tại tạm dừng và quyền thực thi được trả lại để Node có thể xử lý công việc khác. Khi Promise settle, phần continuation được schedule qua microtask mechanism.
>
> Nhưng `await` không làm synchronous CPU computation trở thành non-blocking và không tự tạo thread.

### `await` có block thread không?

> Await một asynchronous Promise không block main thread trong lúc chờ. Nhưng code synchronous chạy trước hoặc sau `await` vẫn có thể block.

---

# 6. Promise.all

## Bài nói

> Em dùng `Promise.all` khi các asynchronous task độc lập và muốn bắt đầu chúng mà không chờ từng task tuần tự. Nhưng `Promise.all` không có concurrency limit.
>
> Đây là điểm em từng gặp trong production: nếu tạo Promise cho rất nhiều shop cùng lúc, database queries cũng được phát ra đồng thời và có thể làm cạn connection pool. Vì vậy với tập dữ liệu lớn em dùng bounded concurrency thay vì `Promise.all` toàn bộ array.

### Nếu một Promise reject?

> `Promise.all` reject khi một Promise reject. Các operation đã được start không tự động bị cancel chỉ vì aggregate Promise reject.

---

# 7. Streams & Backpressure

## Bài nói

> Với file hoặc response lớn em ưu tiên stream thay vì đọc toàn bộ dữ liệu vào memory. Stream xử lý dữ liệu theo chunk nên memory footprint ổn định hơn.
>
> Nhưng stream cần backpressure. Nếu producer tạo dữ liệu nhanh hơn consumer xử lý thì buffer có thể tăng. Với Writable stream, khi `write()` trả `false`, producer nên chờ `drain` hoặc sử dụng `pipe/pipeline` để framework phối hợp flow tốt hơn.

### Các loại Stream?

> Readable, Writable, Duplex và Transform.

---

# 8. Worker Threads vs Child Process

> Worker Threads phù hợp CPU-bound JavaScript và có thể chia sẻ memory thông qua cơ chế như SharedArrayBuffer. Child Process có process/memory space riêng, isolation mạnh hơn nhưng communication overhead thường cao hơn.

---

# 9. Memory Leak

## Bài nói

> Nếu memory của Node process tăng dần và không giảm, em sẽ kiểm tra trước xem đó là traffic/load bình thường hay retained objects thực sự. Những nguyên nhân em nghĩ tới gồm cache không giới hạn, event listener không cleanup, timer, global Map/array hoặc closure giữ reference tới object lớn.
>
> Khi debug em theo dõi heap usage theo thời gian, dùng heap snapshot/profiler và so sánh retained objects thay vì chỉ restart process.

---

# 10. Graceful Shutdown

> Khi process nhận SIGTERM, em không muốn kill ngay. Em ngừng nhận traffic mới, cho in-flight requests hoàn thành trong timeout hợp lý, đóng DB/Redis connections, dừng consumer/job cần thiết, flush telemetry rồi exit. Cần có timeout cuối để tránh process treo mãi.

---

# Rapid-fire questions

### Node single-threaded nghĩa là gì?
JavaScript execution chủ yếu chạy trên một main thread; runtime vẫn dùng OS async mechanisms và worker pool cho một số operation.

### `async/await` có tạo thread?
Không.

### Promise có phải thread?
Không.

### CPU-heavy có vấn đề gì?
Block Event Loop nếu chạy lâu trên main JS thread.

### Stream để làm gì?
Xử lý dữ liệu theo chunk, tránh load toàn bộ vào memory.

### Backpressure là gì?
Cơ chế điều tiết khi producer nhanh hơn consumer.

## Cách nhớ

`V8 → main JS thread → libuv/OS → Event Loop → microtask → CPU blocking → Worker → Stream`
