# Node.js Interview Notes

## 1. Node.js là gì?

Node.js là JavaScript runtime chạy trên V8. Node.js phù hợp với các hệ thống **I/O-bound** và cần xử lý nhiều kết nối đồng thời nhờ kiến trúc **event-driven**, **non-blocking I/O** và **Event Loop**.

Điểm quan trọng:

- JavaScript chủ yếu chạy trên một **main thread**.
- Node.js không có nghĩa là toàn bộ runtime chỉ có một thread.
- I/O có thể được xử lý bởi OS hoặc libuv.
- Một số tác vụ sử dụng **libuv thread pool**.
- CPU-heavy JavaScript chạy lâu trên main thread có thể block Event Loop.

### Node.js phù hợp với

- REST API / GraphQL API.
- Realtime application: WebSocket, chat, notification.
- Proxy / API Gateway.
- Streaming.
- Các service có nhiều I/O: DB, Redis, external API, network.

### Node.js kém phù hợp khi nào?

CPU-bound workload nặng nếu thực hiện trực tiếp trên main thread, ví dụ image/video processing, computation lớn hoặc parsing rất nặng.

Giải pháp có thể là Worker Threads, background workers, job queue hoặc tách thành service riêng.

---

## 2. Event Loop

### Event Loop là gì?

Event Loop là cơ chế cho phép Node.js xử lý nhiều asynchronous operations mà không cần tạo một JavaScript thread cho mỗi request.

JavaScript callback chủ yếu chạy trên main thread. Khi gặp asynchronous operation, Node.js có thể giao công việc cho OS hoặc libuv. Khi công việc hoàn thành, callback sẽ được đưa vào queue phù hợp để Event Loop xử lý khi main thread sẵn sàng.

```text
JavaScript / Main Thread
          |
          | async operation
          v
   Node.js APIs / libuv
          |
          v
 OS / libuv thread pool
          |
          | operation completed
          v
      callback queue
          |
          v
       Event Loop
          |
          v
 JavaScript / Main Thread
```

### Các phase quan trọng

Có thể nhớ simplified flow:

```text
   timers
      |
      v
pending callbacks
      |
      v
     poll
      |
      v
     check
      |
      v
close callbacks
      |
      +----> next iteration
```

#### timers

Xử lý callback của:

- `setTimeout()`
- `setInterval()`

Timer không đảm bảo callback chạy chính xác tại thời điểm delay kết thúc. Delay chỉ thể hiện thời điểm callback **có thể bắt đầu đủ điều kiện** để được xử lý.

#### pending callbacks

Thực thi một số callback I/O bị deferred từ vòng Event Loop trước.

#### poll

Một trong những phase quan trọng nhất:

- Nhận các I/O events mới.
- Thực thi I/O callbacks phù hợp.
- Có thể chờ I/O nếu chưa có công việc cần chạy ngay.

#### check

`setImmediate()` callbacks được thực thi ở phase này.

#### close callbacks

Xử lý một số close events, ví dụ socket emit event `close`.

> Interview: Không nhất thiết phải thuộc toàn bộ internal details của libuv, nhưng nên hiểu rõ `timers`, `poll`, `check` và microtasks.

---

## 3. Microtasks

Promise callback không hoạt động giống timer callback.

```js
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

console.log('4');
```

Kết quả:

```text
1
4
3
2
```

Simplified mental model:

```text
Synchronous JavaScript
        |
        v
    Microtasks
        |
        v
Event Loop callbacks
```

Promise handlers (`then`, `catch`, `finally`) được xử lý qua microtask queue nên trong ví dụ trên Promise callback chạy trước timer callback.

---

## 4. process.nextTick()

Node.js có `process.nextTick()` queue riêng.

```js
process.nextTick(() => console.log('nextTick'));

Promise.resolve().then(() => console.log('promise'));
```

Simplified ordering:

```text
Current JavaScript
       |
       v
process.nextTick queue
       |
       v
Promise microtasks
       |
       v
Event Loop continues
```

Trong Node.js, `process.nextTick()` callbacks được ưu tiên trước Promise microtasks tại các điểm Node drain các queue này.

Không nên enqueue `process.nextTick()` vô hạn vì Event Loop có thể không có cơ hội quay lại xử lý I/O — thường gọi là **I/O starvation**.

---

## 5. setTimeout(0) vs setImmediate()

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
```

Không nên trả lời rằng `setImmediate()` **luôn** chạy trước `setTimeout(0)`.

- `setTimeout()` được xử lý trong timers phase khi timer đủ điều kiện.
- `setImmediate()` được xử lý trong check phase.
- Thứ tự thực tế có thể phụ thuộc vào context và trạng thái Event Loop.
- Khi được schedule bên trong một I/O callback, `setImmediate()` thường có ordering dễ dự đoán hơn so với timer.

---

## 6. libuv và Thread Pool

Một hiểu nhầm phổ biến:

> Node.js single-threaded nên tất cả I/O đều chạy trong thread pool.

Điều này **không đúng**.

Nhiều network I/O được OS xử lý thông qua cơ chế asynchronous event notification. libuv thread pool được dùng cho một số operation không thể hoặc không thuận tiện xử lý hoàn toàn bằng OS async mechanism.

Một số ví dụ thường liên quan thread pool:

- File system APIs.
- Một số DNS operations như `dns.lookup()`.
- Crypto operations.
- zlib compression.

Default libuv thread pool size thường là **4** và có thể cấu hình bằng environment variable:

```bash
UV_THREADPOOL_SIZE=8
```

### Interview trap

**Node.js single-threaded nghĩa là gì?**

JavaScript execution chủ yếu chạy trên một main thread. Nhưng Node.js runtime vẫn có OS async APIs, libuv thread pool và có thể sử dụng Worker Threads.

---

## 7. CPU-bound và Event Loop blocking

CPU-heavy synchronous JavaScript có thể block Event Loop.

```js
app.get('/heavy', (req, res) => {
  let result = 0;

  for (let i = 0; i < 10_000_000_000; i++) {
    result += i;
  }

  res.send(String(result));
});
```

Trong lúc computation chạy:

```text
Request A
   |
CPU-heavy JavaScript
   |
MAIN THREAD BLOCKED
   |
   +-- Request B waiting
   +-- Request C waiting
   +-- timers waiting
   +-- I/O callbacks waiting
```

Các hướng xử lý:

- Worker Threads.
- Background worker.
- Job queue.
- Separate service.
- Chia nhỏ computation nếu bài toán cho phép.

---

## 8. async/await

`async/await` là syntax được xây dựng trên Promise giúp asynchronous code dễ đọc và maintain hơn.

Nó **không**:

- Tạo thread mới.
- Tự động chuyển CPU-heavy work sang background.
- Biến synchronous CPU work thành non-blocking.

```js
async function heavy() {
  let result = 0;

  for (let i = 0; i < 10_000_000_000; i++) {
    result += i;
  }

  return result;
}

await heavy();
```

Loop trên vẫn block main thread.

### await làm gì?

Khi `await` một Promise chưa settle, execution của async function được suspend. Main thread có thể tiếp tục xử lý công việc khác. Khi Promise settle, phần còn lại của async function được schedule để tiếp tục qua Promise/microtask mechanism.

---

## 9. Promise.all và concurrency

`Promise.all()` phù hợp khi các task độc lập và có thể chạy concurrent.

```js
const [user, orders, settings] = await Promise.all([
  getUser(),
  getOrders(),
  getSettings(),
]);
```

Thay vì:

```js
const user = await getUser();
const orders = await getOrders();
const settings = await getSettings();
```

Nếu ba operation độc lập, `Promise.all()` có thể giảm tổng latency.

### Nhưng Promise.all có thể gây overload

```js
await Promise.all(
  shops.map(shop => processShop(shop))
);
```

Nếu có hàng nghìn shop, code có thể đồng thời tạo rất nhiều:

- DB queries.
- HTTP requests.
- Redis operations.
- File operations.

Hậu quả có thể là:

- DB connection pool exhausted.
- API rate limit.
- Memory tăng.
- Timeout.

Do đó cần **concurrency limiting**.

Conceptually:

```text
1000 tasks
    |
    v
concurrency limit = 10
    |
    +--> task 1
    +--> task 2
    +--> ...
    +--> task 10

Task hoàn thành -> lấy task tiếp theo
```

### Promise.all fail-fast

Nếu một Promise reject, `Promise.all()` reject ngay với error đó. Các operation khác đã được bắt đầu không tự động bị cancel.

Nếu cần thu kết quả cả success lẫn failure, cân nhắc `Promise.allSettled()`.

---

## 10. Concurrency vs Parallelism

Hai khái niệm dễ bị nhầm.

### Concurrency

Nhiều task có thể cùng **in progress** và xen kẽ thời gian chờ.

Ví dụ Node.js xử lý nhiều HTTP requests đang chờ DB/network.

### Parallelism

Nhiều task thực sự execute **cùng một thời điểm** trên nhiều CPU cores/threads.

Ví dụ Worker Threads chạy computation trên nhiều threads.

```text
Concurrency:
Task A: RUN ---- WAIT -------- RUN
Task B: ---- RUN ---- WAIT ------- RUN

Parallelism:
CPU 1: Task A ====================
CPU 2: Task B ====================
```

Node.js rất mạnh về concurrency cho I/O-bound workloads dù JavaScript execution mặc định chủ yếu nằm trên một main thread.

---

## 11. Streams

Streams cho phép xử lý dữ liệu theo **chunk** thay vì load toàn bộ dữ liệu vào memory.

Ví dụ không tốt với file rất lớn:

```js
const data = await fs.promises.readFile('large-file.zip');
```

Toàn bộ file phải nằm trong memory.

Streaming:

```js
const stream = fs.createReadStream('large-file.zip');
stream.pipe(res);
```

Dữ liệu được đọc và gửi theo từng chunk.

### Các loại Stream

- **Readable** — đọc dữ liệu.
- **Writable** — ghi dữ liệu.
- **Duplex** — vừa đọc vừa ghi.
- **Transform** — đọc, transform rồi ghi output.

Ví dụ Transform stream: gzip compression.

### Lợi ích

- Memory footprint thấp hơn.
- Có thể bắt đầu xử lý trước khi toàn bộ dữ liệu được load.
- Phù hợp file upload/download, proxy, video hoặc large datasets.

---

## 12. Backpressure

Backpressure xảy ra khi **producer tạo dữ liệu nhanh hơn consumer có thể xử lý**.

```text
Producer
   |
   | FAST
   v
 Buffer
   |
   | SLOW
   v
Consumer
```

Nếu không kiểm soát, buffer có thể tăng liên tục và dẫn đến memory pressure hoặc OOM.

Streams của Node.js có cơ chế hỗ trợ backpressure.

Ví dụ với writable stream:

```js
const canContinue = writable.write(chunk);

if (!canContinue) {
  await once(writable, 'drain');
}
```

Khi `write()` trả về `false`, producer nên chờ consumer xử lý bớt dữ liệu trước khi tiếp tục ghi.

`pipe()` / `pipeline()` thường giúp quản lý flow này thuận tiện hơn.

---

## 13. Worker Threads

Worker Threads phù hợp với **CPU-bound JavaScript**.

Ví dụ:

- Heavy parsing.
- Image processing bằng JS/native bindings phù hợp.
- Encryption/computation.
- Data transformation lớn.

```text
Main Thread
    |
    +------ Worker 1
    |
    +------ Worker 2
    |
    +------ Worker 3
```

Worker có JavaScript execution context riêng và có thể chạy song song trên CPU cores khác nhau.

### Worker Threads không phải lựa chọn mặc định cho I/O

Nếu task chỉ là gọi DB/API/network thì asynchronous I/O của Node.js thường đã phù hợp. Tạo Worker Thread cho mỗi I/O request có thể làm architecture phức tạp hơn mà không đem lại lợi ích tương ứng.

---

## 14. Worker Threads vs Child Process vs Cluster

### Worker Threads

- Nhiều threads trong cùng process.
- Phù hợp CPU-heavy computation.
- Có thể chia sẻ memory thông qua `SharedArrayBuffer` khi cần.

### Child Process

- Tạo process riêng.
- Memory/process isolation tốt hơn.
- Có thể chạy command hoặc chương trình khác.
- Communication thường qua IPC/stdin/stdout.

### Cluster

Cho phép chạy nhiều Node.js processes để tận dụng nhiều CPU cores cho server workload. Trong production hiện đại, việc scale nhiều process/instance cũng thường được quản lý ở process manager, container hoặc orchestration layer.

---

## 15. Memory leak thường gặp

Memory leak là trường hợp object không còn cần thiết nhưng vẫn còn reachable reference nên Garbage Collector không thể giải phóng.

Các nguyên nhân thường gặp:

- Cache không giới hạn.
- Event listener không cleanup.
- Timer giữ reference.
- Global collection tăng mãi.
- Closure giữ object lớn.
- Request/session data bị giữ ngoài lifecycle cần thiết.

Ví dụ:

```js
const cache = {};

app.get('/user/:id', async (req, res) => {
  cache[req.params.id] = await getUser(req.params.id);
  res.json(cache[req.params.id]);
});
```

Nếu cache không có eviction policy, memory có thể tăng liên tục.

### Cách debug

- Theo dõi `process.memoryUsage()`.
- Heap snapshot.
- Chrome DevTools / Node inspector.
- So sánh retained objects giữa các heap snapshots.
- Theo dõi memory trend trong production.

---

## 16. Graceful Shutdown

Khi application nhận signal shutdown như `SIGTERM`, không nên exit ngay nếu vẫn còn request hoặc connection đang xử lý.

Flow phổ biến:

```text
SIGTERM
   |
   v
Stop accepting new traffic
   |
   v
Wait for in-flight requests
   |
   v
Close DB / Redis / queue connections
   |
   v
Flush required telemetry/logs
   |
   v
Exit
```

Ví dụ simplified:

```js
process.on('SIGTERM', async () => {
  server.close(async () => {
    await prisma.$disconnect();
    await redis.quit();
    process.exit(0);
  });
});
```

Production implementation thường cần timeout/fallback để process không chờ vô hạn.

---

# Câu hỏi phỏng vấn

## Q1. Node.js single-threaded nghĩa là gì?

JavaScript execution chủ yếu chạy trên một main thread, nhưng Node.js runtime không hoàn toàn single-threaded. Node có asynchronous OS APIs, libuv thread pool và có thể sử dụng Worker Threads.

## Q2. Tại sao Node.js xử lý được nhiều request đồng thời?

Vì Node.js không block main thread để chờ phần lớn I/O. Khi request đang chờ DB/network/I/O, Event Loop có thể tiếp tục xử lý công việc khác. Khi operation hoàn thành, callback/continuation được schedule để JavaScript xử lý.

## Q3. Event Loop là gì?

Event Loop điều phối việc thực thi asynchronous callbacks trên main JavaScript thread. Nó làm việc cùng OS và libuv để Node.js có thể xử lý nhiều I/O operations concurrent mà không cần một JavaScript thread cho mỗi request.

## Q4. Các Event Loop phase quan trọng?

Nên nhớ ít nhất:

- timers
- poll
- check

Ngoài ra còn pending callbacks và close callbacks.

## Q5. Promise và setTimeout cái nào chạy trước?

Trong ví dụ cơ bản khi cả hai được schedule trong cùng synchronous execution, Promise microtask được xử lý trước timer callback.

Không nên biến quy tắc này thành một khẳng định tuyệt đối cho mọi asynchronous context; cần xét nơi callback được schedule.

## Q6. process.nextTick khác Promise như thế nào?

Node.js có nextTick queue riêng. `process.nextTick()` callbacks được drain trước Promise microtasks tại các điểm tương ứng. Lạm dụng nextTick có thể gây I/O starvation.

## Q7. setImmediate và setTimeout(0) khác gì?

`setImmediate()` chạy ở check phase; `setTimeout()` chạy ở timers phase khi timer đủ điều kiện. Không đảm bảo `setImmediate()` luôn chạy trước timer trong mọi context.

## Q8. async/await có tạo thread mới không?

Không. `async/await` là syntax trên Promise và không tự tạo thread mới.

## Q9. await có block Event Loop không?

`await` một asynchronous Promise không block toàn bộ Event Loop. Nó suspend async function hiện tại và cho phép main thread làm công việc khác.

Nhưng nếu code trước Promise completion là CPU-heavy synchronous work thì phần CPU-heavy đó vẫn block Event Loop.

## Q10. Promise.all có tạo thread không?

Không. `Promise.all()` chỉ phối hợp nhiều Promise. Các underlying operations có thể được OS, libuv hoặc code khác xử lý concurrent tùy loại operation.

## Q11. Khi nào không nên Promise.all toàn bộ array?

Khi số lượng task lớn hoặc resource phía dưới có giới hạn, ví dụ DB connection pool hoặc API rate limit. Khi đó nên giới hạn concurrency.

## Q12. Concurrency khác parallelism thế nào?

Concurrency là nhiều task cùng in progress; parallelism là nhiều task thực sự execute đồng thời. Node.js Event Loop rất hiệu quả với concurrency cho I/O, còn Worker Threads có thể cung cấp parallelism cho CPU work.

## Q13. Tất cả I/O có chạy trong libuv thread pool không?

Không. Nhiều network I/O dựa vào asynchronous mechanisms của OS. Thread pool chủ yếu hỗ trợ một số operation như filesystem, một số DNS, crypto và zlib.

## Q14. CPU-heavy code ảnh hưởng Node.js thế nào?

Nếu chạy synchronous trên main thread, nó block Event Loop khiến request, timer và callbacks khác phải chờ.

## Q15. Khi nào dùng Worker Threads?

Khi có CPU-bound JavaScript computation đủ nặng để ảnh hưởng Event Loop. Không cần dùng Worker Threads chỉ để gọi DB/API thông thường.

## Q16. Stream có lợi gì?

Stream xử lý dữ liệu theo chunk nên giảm memory footprint và cho phép bắt đầu xử lý dữ liệu trước khi toàn bộ payload được load vào memory.

## Q17. Backpressure là gì?

Là cơ chế kiểm soát flow khi producer nhanh hơn consumer, giúp tránh buffer/memory tăng không kiểm soát.

## Q18. Làm sao phát hiện Event Loop đang bị block?

Có thể theo dõi event-loop delay/latency, CPU usage, request latency và profiling. Nếu CPU cao và event-loop delay tăng, cần tìm synchronous/CPU-heavy work đang giữ main thread.

## Q19. Memory leak thường đến từ đâu?

Cache không eviction, event listeners không cleanup, timers, global collections, closures giữ object lớn hoặc reference sống lâu hơn lifecycle cần thiết.

## Q20. Graceful shutdown để làm gì?

Để service ngừng nhận traffic mới nhưng hoàn thành công việc đang xử lý và đóng resources như HTTP server, DB, Redis hoặc queue một cách an toàn trước khi process exit.

---

# Event Loop — câu trả lời mẫu 30–45 giây

> Node.js executes JavaScript primarily on a single main thread. When asynchronous operations such as network or file I/O occur, Node can delegate work to the operating system or libuv. When the operation completes, its callback becomes eligible to be processed by the Event Loop.
>
> The Event Loop moves through phases such as timers, poll and check. Node also processes microtasks such as Promise callbacks and has a higher-priority `process.nextTick` queue.
>
> Because JavaScript callbacks execute on the main thread, CPU-intensive synchronous code can block the Event Loop. For CPU-heavy workloads we can use Worker Threads or move work to background workers or separate services.

---

# Cheat Sheet

- Node.js mạnh với **I/O-bound** workloads.
- JavaScript execution chủ yếu chạy trên **main thread**.
- Node.js runtime không hoàn toàn single-threaded.
- Event Loop giúp Node xử lý nhiều asynchronous operations concurrent.
- Event Loop: nhớ `timers -> poll -> check`.
- Promise handlers là **microtasks**.
- `process.nextTick()` có priority cao hơn Promise microtasks trong Node.js.
- `setImmediate()` thuộc check phase.
- `setTimeout()` thuộc timers phase.
- Không phải mọi I/O đều chạy trong libuv thread pool.
- CPU-heavy synchronous code có thể block Event Loop.
- `async/await` không tạo thread mới.
- `await` asynchronous I/O không đồng nghĩa block Event Loop.
- `Promise.all()` không tạo thread mới.
- `Promise.all()` với quá nhiều tasks có thể overload DB/API.
- Cần concurrency limiting khi fan-out lớn.
- Stream xử lý dữ liệu lớn theo chunk.
- Backpressure bảo vệ consumer/memory khi producer quá nhanh.
- Worker Threads phù hợp CPU-bound computation.
- Memory leak thường do reference sống quá lâu.
- Graceful shutdown: stop traffic -> drain requests -> close resources -> exit.
