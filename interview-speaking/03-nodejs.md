# 03 — Node.js & Event Loop

Mục tiêu: **nói thành câu chuyện**, không đọc thuộc một chuỗi keyword. Thuật ngữ quan trọng vẫn giữ lại nhưng giải thích ngay khi xuất hiện.

---

# 1. Node.js là gì?

## 💬 Bài nói

> Node.js là môi trường giúp em chạy JavaScript bên ngoài browser, thường dùng để xây backend/API. Bên dưới Node dùng **V8** *(JavaScript engine thực thi code JavaScript)* và **libuv** *(thư viện hỗ trợ Event Loop và nhiều cơ chế I/O bất đồng bộ)*.
>
> Điểm mạnh của Node.js là xử lý tốt những hệ thống có nhiều thời gian chờ I/O, ví dụ chờ database, chờ Shopify API hoặc network response. Trong lúc một operation đang chờ I/O, main JavaScript thread không nhất thiết phải đứng yên chỉ để đợi kết quả đó.
>
> JavaScript application chủ yếu chạy trên một main thread. Nhưng nói “Node.js single-threaded” không có nghĩa toàn bộ runtime chỉ có đúng một thread. Node còn dùng cơ chế asynchronous I/O của hệ điều hành và libuv worker pool cho một số operation.
>
> Điểm cần chú ý là nếu có đoạn JavaScript tính toán CPU quá lâu trên main thread thì những callback/request khác phải chờ và server có thể phản hồi chậm.

## 🧾 Thuật ngữ

### **Runtime** *(môi trường cung cấp những thứ cần thiết để code chạy)*

Browser là một JavaScript runtime. Node.js cũng là runtime nhưng có API phía server như filesystem, network, process...

### **V8** *(JavaScript engine)*

V8 chịu trách nhiệm parse/compile/execute JavaScript. Node.js dùng V8 nhưng Node không chỉ có V8.

### **I/O** *(Input/Output — thao tác đọc/ghi hoặc giao tiếp với hệ thống bên ngoài CPU)*

Ví dụ đọc file, query database, gọi API, network socket.

### **I/O-bound** *(task dành phần lớn thời gian để chờ I/O)*

Ví dụ API gọi PostgreSQL rồi chờ query trả kết quả.

### **CPU-bound** *(task dành phần lớn thời gian để CPU tính toán)*

Ví dụ vòng lặp tính toán rất lớn, xử lý ảnh, encode/compress nặng.

---

# 2. Synchronous và Asynchronous

## **Synchronous** *(đồng bộ — operation hiện tại hoàn thành rồi code mới đi tiếp)*

```js
const data = readFileSync('large.txt');
console.log(data.length);
```

Trong thời gian `readFileSync` chạy, JavaScript thread bị giữ ở đó.

## **Asynchronous** *(bất đồng bộ — bắt đầu operation rồi có thể tiếp tục xử lý việc khác trong thời gian chờ)*

```js
const data = await fs.promises.readFile('large.txt');
```

Khi Promise đang chờ I/O, main thread không phải bận chỉ để ngồi chờ file.

⚠️ **Dễ bị bắt bẻ:**

> “Async nghĩa là chạy trên thread khác.”

Câu này sai trong nhiều trường hợp.

✅ **Cách nói an toàn:**

> Asynchronous nghĩa là flow không cần block JavaScript thread trong thời gian chờ operation. Cách operation được thực hiện bên dưới có thể là OS async mechanism hoặc worker pool tùy loại công việc.

---

# 3. Event Loop

## 💬 Bài nói 60–90 giây

> Em hình dung **Event Loop** là cơ chế giúp Node điều phối khi nào những callback hoặc phần code bất đồng bộ đã sẵn sàng được chạy lại trên JavaScript thread.
>
> Ví dụ request cần query database. Node gửi network request tới database. Trong thời gian database đang xử lý, JavaScript thread có thể tiếp tục phục vụ công việc khác.
>
> Khi database trả kết quả, runtime nhận được event hoàn thành. Phần code xử lý kết quả sẽ được schedule để chạy khi tới lượt và khi JavaScript thread sẵn sàng.
>
> Nhờ mô hình này, Node có thể quản lý nhiều I/O operation cùng lúc dù JavaScript application chủ yếu chạy trên một main thread.

## 📌 Sơ đồ

```text
Request A
   ↓
Query database ────────────────┐
                               │ DB đang xử lý
Main JS thread làm việc khác   │
                               ↓
                         DB trả kết quả
                               ↓
                     Event Loop / scheduling
                               ↓
                    JS xử lý kết quả
```

## 🧾 Thuật ngữ

### **Callback** *(function được đăng ký để chạy khi một công việc/sự kiện tới lúc xử lý)*

### **Scheduling** *(quyết định khi nào công việc đã sẵn sàng được đưa tới lượt chạy)*

### **Queue** *(nơi/cơ chế các công việc sẵn sàng chờ được xử lý)*

Node có nhiều queue/cơ chế scheduling khác nhau, không phải mọi callback đều nằm trong một queue duy nhất.

---

# 4. Event Loop phases

Nếu interviewer hỏi sâu, có thể nói:

> Event Loop của Node/libuv có nhiều phase. Những phase em thường nhớ và giải thích được là **timers**, **poll**, **check** và **close callbacks**. Em ưu tiên hiểu mỗi phase làm gì hơn là đọc thuộc tên.

### **Timers**

Xử lý callback của timer khi timer đã đạt điều kiện thời gian.

`setTimeout(fn, 1000)` không có nghĩa callback chắc chắn chạy đúng 1000ms. Nó chỉ không chạy trước delay tối thiểu; lúc đó main thread có thể vẫn đang bận.

### **Poll**

Có thể hiểu đơn giản là phase quan trọng liên quan tới nhận và xử lý I/O events/callbacks đã sẵn sàng.

### **Check**

`setImmediate()` callback được xử lý ở check phase.

### **Close callbacks**

Liên quan tới một số callback khi resource/socket/handle được đóng.

⚠️ **Dễ bị bắt bẻ:**

> “Event Loop chỉ có timers → poll → check.”

✅ **Cách nói an toàn:**

> Đó là mental model rút gọn em dùng để nhớ. Thực tế Event Loop có thêm các phase/cơ chế khác.

---

# 5. Promise, Microtask và `process.nextTick`

## **Microtask** *(nhóm công việc có ưu tiên scheduling cao hơn nhiều callback thông thường)*

Promise `.then()` và phần tiếp tục sau `await` dùng cơ chế microtask.

```js
console.log('A');

setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));

console.log('B');
```

Kết quả cơ bản:

```text
A
B
promise
timeout
```

Vì code synchronous chạy trước, sau đó Promise microtask được xử lý trước timer callback trong ví dụ này.

### `process.nextTick()` là gì?

Đây là API riêng của Node để schedule callback chạy rất sớm sau operation hiện tại. Node xử lý nextTick queue với độ ưu tiên cao.

### **Starvation** *(một nhóm task cứ được ưu tiên liên tục khiến nhóm khác không có cơ hội chạy)*

Nếu code liên tục tạo thêm `process.nextTick`, I/O có thể bị trì hoãn.

⚠️ **Dễ bị bắt bẻ:**

> “Promise luôn chạy trước mọi thứ.”

✅ **Cách nói an toàn:**

> Promise continuation dùng microtask queue và thường được xử lý trước nhiều callback event-loop thông thường sau khi current JavaScript hoàn thành. Nhưng em không dùng từ “luôn” cho mọi context.

---

# 6. `setTimeout(0)` vs `setImmediate()`

## 💬 Cách nói

> `setTimeout(fn, 0)` không có nghĩa chạy ngay. Nó đăng ký timer với delay tối thiểu. `setImmediate()` được xử lý ở check phase. Em không nói một cái luôn chạy trước cái kia trong mọi trường hợp vì thứ tự còn phụ thuộc context và trạng thái Event Loop.

📌 Khi schedule từ một I/O callback, `setImmediate()` thường có flow dễ dự đoán hơn theo poll → check.

---

# 7. libuv và Worker Pool

## **libuv** *(thư viện Node dùng cho Event Loop, abstraction I/O và worker pool)*

Node chạy trên nhiều OS khác nhau. libuv giúp cung cấp interface tương đối thống nhất cho những cơ chế I/O đó.

## **Worker pool** *(một nhóm worker threads được runtime giữ sẵn để xử lý một số operation)*

Không phải mỗi async operation tạo một thread mới.

Những operation thường được nhắc tới khi nói về libuv worker pool gồm:

- nhiều filesystem APIs;
- `dns.lookup`;
- một số crypto operations;
- zlib/compression operations.

### Network request có dùng một worker thread cho mỗi request không?

> Không. Network socket I/O thường dựa vào asynchronous event notification của operating system, chứ không phải một request tương ứng một thread trong libuv pool.

### Default worker pool size?

> Thông thường mặc định là 4 và có thể điều chỉnh bằng `UV_THREADPOOL_SIZE`. Nhưng tăng thread không tự động làm mọi workload nhanh hơn.

### **Contention** *(nhiều task cùng tranh một resource giới hạn)*

Ví dụ nhiều task tranh CPU, database connection hoặc lock. Concurrency quá cao đôi khi làm hệ thống chậm hơn.

---

# 8. `async/await`

## 💬 Bài nói

> `async/await` là cách viết code dựa trên Promise giúp flow bất đồng bộ dễ đọc hơn. Khi em `await` một Promise đang chờ, function hiện tại tạm dừng ở vị trí đó nhưng main JavaScript thread không phải đứng yên chỉ để chờ Promise.
>
> Khi Promise hoàn thành, phần code sau `await` được schedule để tiếp tục chạy.
>
> Nhưng `async/await` không tự tạo thread và cũng không biến một vòng lặp CPU-heavy thành non-blocking.

### **Pending Promise** *(Promise chưa có kết quả cuối)*

### **Fulfilled** *(Promise hoàn thành thành công)*

### **Rejected** *(Promise kết thúc với lỗi)*

### **Settled** *(Promise đã fulfilled hoặc rejected)*

⚠️ **Dễ bị bắt bẻ:**

> “`await` không block.”

✅ **Cách nói an toàn:**

> `await` một Promise thực sự asynchronous không giữ main thread đứng chờ. Nhưng synchronous JavaScript chạy trước hoặc sau `await` vẫn có thể block.

---

# 9. `Promise.all` và Concurrency

## 💬 Bài nói

> `Promise.all` hữu ích khi em có nhiều asynchronous task độc lập. Thay vì chờ task 1 xong rồi mới bắt đầu task 2, em có thể start nhiều task và chờ tất cả hoàn thành.
>
> Tuy nhiên `Promise.all` không tự giới hạn số task đang chạy. Nếu em map 1.000 shop thành 1.000 Promise gọi database/API thì application có thể tạo áp lực rất lớn lên database connection pool hoặc API rate limit.
>
> Với collection lớn em thường dùng **bounded concurrency** — tức là vẫn chạy nhiều task cùng lúc nhưng đặt giới hạn rõ ràng, ví dụ tối đa 5 hoặc 10 shop đang active.

### **Concurrency** *(nhiều task cùng đang trong quá trình xử lý)*

### **Parallelism** *(nhiều task thực sự execute cùng một thời điểm trên nhiều CPU/thread)*

Concurrency không bắt buộc phải là parallelism.

### **Connection pool** *(nhóm database connections được tái sử dụng)*

Nếu pool có 20 connection nhưng hàng trăm query cùng cần connection thì nhiều query phải chờ và có thể timeout.

---

# 10. CPU-bound và Worker Threads

## 📌 Ví dụ

```js
app.get('/heavy', (req, res) => {
  let total = 0;
  for (let i = 0; i < 10_000_000_000; i++) {
    total += i;
  }
  res.send(String(total));
});
```

Trong lúc vòng lặp chạy, main JS thread bị chiếm.

## **Worker Thread** *(thread khác trong Node process có thể chạy JavaScript riêng)*

Phù hợp khi thật sự có CPU-heavy JavaScript cần đưa ra khỏi main thread.

### Khi nào không cần Worker Thread?

> Nếu task chủ yếu là chờ database/API thì Worker Thread thường không giải quyết đúng vấn đề. Node vốn đã xử lý I/O concurrency tốt; thêm worker có thể chỉ làm kiến trúc phức tạp hơn.

### Worker Thread vs Child Process

- **Worker Thread**: cùng process, có thể chia sẻ memory bằng cơ chế phù hợp.
- **Child Process**: process riêng, memory space riêng, isolation mạnh hơn nhưng communication nặng hơn.

---

# 11. Stream & Backpressure

## 💬 Bài nói

> Với file lớn em ưu tiên stream khi không cần load toàn bộ file vào memory. Stream xử lý dữ liệu theo từng phần nhỏ, gọi là chunk.
>
> Tuy nhiên nếu phía đọc tạo dữ liệu nhanh hơn phía ghi hoặc network xử lý thì buffer có thể tăng. Lúc đó cần **backpressure** — cơ chế làm phía tạo dữ liệu chậm lại để phía nhận theo kịp.

### **Chunk** *(một phần nhỏ của dữ liệu lớn)*

### **Producer** *(phía tạo/đọc dữ liệu)*

### **Consumer** *(phía nhận/xử lý dữ liệu)*

### **Backpressure** *(điều tiết producer khi consumer xử lý không kịp)*

Với Writable stream, `write()` có thể trả `false`. Khi đó producer nên chờ `drain`, hoặc sử dụng `pipe()`/`pipeline()` để Node phối hợp flow.

📌 Ví dụ: đọc file 5GB từ disk rất nhanh nhưng upload qua network chậm hơn. Nếu cứ đọc không giới hạn, memory buffer có thể tăng.

---

# 12. Memory Leak

## 💬 Bài nói

> Nếu memory của Node process tăng dần và không giảm, em không kết luận ngay là memory leak. Em xem trước traffic/load, heap usage theo thời gian và object nào đang bị giữ lại.
>
> Các nguyên nhân thường gặp gồm cache không giới hạn, event listener không cleanup, timer, global Map/array hoặc closure giữ reference tới object lớn.
>
> Khi debug em dùng heap snapshot/profiler để xem object nào vẫn bị giữ sau nhiều vòng garbage collection.

### **Garbage Collection / GC** *(cơ chế runtime tự thu hồi memory của object không còn được tham chiếu)*

### **Retained object** *(object vẫn còn reference nên GC chưa thể thu hồi)*

⚠️ Memory tăng không tự động đồng nghĩa memory leak.

---

# 13. Graceful Shutdown

## 💬 Bài nói

> Khi process nhận tín hiệu dừng như `SIGTERM`, em không muốn kill ngay nếu đang có request/job quan trọng. Em ngừng nhận traffic mới, cho request đang chạy hoàn thành trong một timeout hợp lý, đóng database/Redis connection, dừng consumer cần thiết rồi exit.

### **Graceful shutdown** *(tắt service có kiểm soát để giảm request/job bị cắt giữa chừng)*

### **In-flight request** *(request đã bắt đầu nhưng chưa hoàn thành)*

---

# 🎯 Rapid-fire follow-up

### Node single-threaded nghĩa là gì?

> JavaScript application chủ yếu execute trên một main thread. Runtime vẫn dùng OS asynchronous mechanisms và worker threads cho một số operation.

### `async/await` có tạo thread mới không?

> Không.

### Promise có phải thread không?

> Không. Promise mô tả kết quả bất đồng bộ và cách continuation được schedule.

### CPU-heavy gây vấn đề gì?

> Nếu chạy synchronous lâu trên main JavaScript thread thì block Event Loop và làm request khác phải chờ.

### Stream lợi gì?

> Xử lý dữ liệu theo chunk, tránh phải load toàn bộ dữ liệu lớn vào memory.

### Backpressure là gì?

> Cơ chế làm producer chậm lại khi consumer xử lý không kịp.

---

# ⚠️ Những câu không nên nói

❌ “Node.js chỉ có một thread.”  
✅ “JavaScript application chủ yếu chạy trên một main thread, nhưng runtime còn có OS async mechanisms và worker pool.”

❌ “Async operation sẽ tạo thread mới.”  
✅ “Async không đồng nghĩa tạo thread; tùy operation mà Node dùng OS async mechanism hoặc worker pool.”

❌ “`setTimeout(0)` chạy ngay.”  
✅ “0 là minimum delay; callback vẫn phải chờ tới lúc timer được xử lý.”

❌ “`Promise.all` chạy parallel.”  
✅ “`Promise.all` start/chờ nhiều asynchronous task; mức parallel thực tế phụ thuộc bản chất operation.”

---

# 📌 Cách nhớ

**V8 → main JS thread → async I/O → Event Loop → Promise/microtask → libuv → CPU blocking → Worker → Stream/backpressure**

Đừng học chuỗi này như định nghĩa. Hãy dùng nó như bản đồ để kể từ trên xuống.