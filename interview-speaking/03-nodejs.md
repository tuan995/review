# 03 — Node.js & Event Loop

> Mục tiêu: có thể nói thành câu chuyện và tự giải thích được từng thuật ngữ nếu interviewer dừng lại hỏi "cái đó là gì?".

# 1. Node.js là gì?

## Bài nói

> Node.js là môi trường giúp em chạy JavaScript bên ngoài browser, thường dùng để xây backend/API. Bên dưới Node.js sử dụng V8 để thực thi JavaScript và libuv để hỗ trợ nhiều phần liên quan tới asynchronous I/O và Event Loop.
>
> Điểm mạnh của Node.js là xử lý các hệ thống có nhiều thao tác phải chờ như gọi database, gọi API hoặc network I/O. Thay vì đứng chờ một request I/O hoàn thành rồi mới làm request tiếp theo, Node có thể đăng ký công việc bất đồng bộ và tiếp tục xử lý việc khác.
>
> JavaScript application chủ yếu chạy trên một main thread. Tuy nhiên "Node.js single-threaded" không có nghĩa toàn bộ Node chỉ có đúng một thread. Runtime còn có OS asynchronous mechanisms và libuv worker pool cho một số loại công việc.
>
> Điểm cần cẩn thận là nếu em chạy JavaScript tính toán CPU quá lâu trên main thread, những callback/request khác không có cơ hội chạy kịp và server có thể phản hồi chậm.

### Runtime là gì?

> Runtime có thể hiểu là môi trường cung cấp những thứ cần thiết để code JavaScript thực thi. Browser là một JavaScript runtime; Node.js cũng là một runtime nhưng cung cấp API phù hợp phía server như filesystem, network, process...

### V8 là gì?

> V8 là JavaScript engine thực thi JavaScript. Node.js dùng V8 nhưng Node.js không chỉ có V8; Node còn có các API và thành phần như libuv để làm việc với I/O và Event Loop.

### I/O-bound là gì?

> I/O-bound nghĩa là phần lớn thời gian của task nằm ở việc chờ input/output thay vì CPU tính toán. Ví dụ chờ PostgreSQL trả query, chờ Shopify API response hoặc chờ đọc file.

### CPU-bound là gì?

> CPU-bound là task tốn phần lớn thời gian để CPU tính toán, ví dụ vòng lặp tính toán rất lớn, encode/compress nặng hoặc xử lý ảnh. Nếu chạy synchronous trên main JS thread thì nó có thể block Event Loop.

---

# 2. Synchronous và Asynchronous

### Synchronous là gì?

> Code synchronous chạy theo thứ tự và operation hiện tại phải hoàn thành trước khi JavaScript tiếp tục dòng tiếp theo.

### Asynchronous là gì?

> Với asynchronous operation, application có thể bắt đầu một công việc cần chờ, sau đó không nhất thiết phải giữ main JavaScript thread đứng yên chờ kết quả. Khi kết quả sẵn sàng, phần code tiếp theo được schedule để chạy sau.

> Asynchronous không đồng nghĩa với "tạo thread mới" và cũng không đồng nghĩa mọi thứ thực sự chạy parallel.

---

# 3. Event Loop

## Bài nói từ đầu đến cuối

> Em hình dung Event Loop là cơ chế giúp Node quyết định khi nào các callback bất đồng bộ có thể quay lại chạy trên JavaScript thread.
>
> Ví dụ request của em cần query database. Node gửi network request tới database. Trong thời gian chờ database trả lời, main JavaScript thread không cần ngồi chờ riêng query đó mà có thể xử lý công việc khác.
>
> Khi I/O có kết quả, runtime nhận được sự kiện hoàn thành. Callback hoặc continuation tương ứng sẽ có cơ hội được Event Loop xử lý khi JavaScript thread sẵn sàng.
>
> Vì vậy Node có thể phục vụ nhiều I/O operation concurrent dù JavaScript application chủ yếu chạy trên một main thread.

```text
Request A → query DB ───────────────┐
                                   │ DB đang xử lý
Main JS thread → xử lý việc khác   │
                                   ↓
                              DB trả kết quả
                                   ↓
                         Event Loop / scheduling
                                   ↓
                       JS xử lý phần tiếp theo
```

### Callback là gì?

> Callback là function được truyền/đăng ký để chạy khi một công việc hoặc sự kiện đến thời điểm xử lý. Ví dụ sau khi I/O hoàn thành, callback có thể được gọi để xử lý kết quả.

### Queue là gì trong ngữ cảnh này?

> Có thể hiểu đơn giản queue là nơi các công việc đã sẵn sàng chờ tới lượt được xử lý. Node thực tế có nhiều queue/cơ chế scheduling khác nhau, nên khi phỏng vấn em không nói rằng tất cả callback đều nằm trong một queue duy nhất.

---

# 4. Event Loop phases

> Nếu interviewer hỏi sâu, em biết Event Loop của Node/libuv đi qua các phase. Những phase em thường nhớ là timers, pending callbacks, poll, check và close callbacks.

### Timers

> Đây là nơi xử lý callback của timer khi timer đã đạt ngưỡng thời gian phù hợp. `setTimeout(fn, 1000)` có nghĩa callback không được chạy trước khoảng delay đó; không đảm bảo đúng chính xác 1000 ms vì lúc đó main thread có thể đang bận.

### Poll

> Poll là phase quan trọng liên quan tới việc nhận/xử lý các I/O events sẵn sàng. Khi nói đơn giản, em nhớ đây là phần Event Loop dành nhiều thời gian để xử lý I/O callbacks/events.

### Check

> `setImmediate()` callback được xử lý ở check phase.

### Close callbacks

> Liên quan tới một số callback đóng resource, ví dụ close event của socket/handle.

### Có cần thuộc mọi phase không?

> Em ưu tiên hiểu flow và các case thực tế hơn là chỉ đọc tên phase. Nếu cần nhớ nhanh: timer → I/O/poll → check, nhưng đây chỉ là mental model rút gọn chứ không phải toàn bộ implementation.

---

# 5. Microtask, Promise và `process.nextTick`

## Microtask là gì?

> Microtask là nhóm công việc có mức ưu tiên scheduling khác callback thông thường. Promise continuation như `.then()`/phần sau `await` sử dụng microtask mechanism. Sau khi JavaScript hiện tại hoàn thành, microtasks được xử lý tại các điểm thích hợp trước khi Event Loop tiếp tục sang nhiều callback thông thường khác.

Ví dụ:

```js
console.log('A');

setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));

console.log('B');
```

Kết quả cơ bản cần nhớ:

```text
A
B
promise
timeout
```

> `A` và `B` là synchronous. Promise continuation là microtask nên được xử lý trước timer callback trong ví dụ này.

### `process.nextTick()` là gì?

> Đây là API riêng của Node để schedule callback chạy rất sớm sau operation hiện tại. Node xử lý nextTick queue với độ ưu tiên cao hơn Promise microtask queue tại các điểm nó drain hai queue này.

### Starvation là gì?

> Starvation nghĩa là một nhóm công việc cứ được ưu tiên liên tục khiến nhóm khác không có cơ hội chạy. Nếu code liên tục schedule thêm `process.nextTick`, Event Loop có thể cứ xử lý nextTick và trì hoãn I/O.

---

# 6. `setTimeout(0)` và `setImmediate()`

> `setTimeout(fn, 0)` không có nghĩa "chạy ngay". Nó đăng ký timer với delay tối thiểu và callback chỉ chạy khi đến lượt timer processing.
>
> `setImmediate()` được schedule cho check phase.
>
> Em không nói `setImmediate` luôn trước hoặc timer luôn trước trong mọi trường hợp, vì thứ tự phụ thuộc context và trạng thái Event Loop. Khi schedule từ một I/O callback, `setImmediate` thường có behavior dễ dự đoán hơn theo flow poll → check.

---

# 7. libuv và Worker Pool

## libuv là gì?

> libuv là thư viện mà Node dùng cho Event Loop và abstraction asynchronous I/O trên nhiều hệ điều hành. Nó giúp Node làm việc với những cơ chế I/O khác nhau của OS theo một interface thống nhất và cũng cung cấp worker thread pool.

## Worker pool là gì?

> Đây là một nhóm worker threads được runtime duy trì để xử lý một số operation không chạy theo cơ chế non-blocking event notification thông thường. Không phải mỗi async operation tạo một thread mới.

### Những gì có thể dùng libuv thread pool?

> Nhiều filesystem APIs, `dns.lookup`, một số crypto và zlib operations là những ví dụ thường gặp.

### Network request có phải mỗi request chiếm một thread pool thread?

> Không. Network socket I/O thường dựa vào asynchronous I/O/event notification của operating system. Đây là lý do không nên mô tả Node là "mỗi async task chạy trên thread pool".

### Default thread pool size?

> Thường mặc định là 4 và có thể cấu hình `UV_THREADPOOL_SIZE`. Nhưng tăng số thread không phải thuốc chữa mọi performance problem vì còn phụ thuộc CPU, loại workload và contention.

### Contention là gì?

> Contention là nhiều task cùng tranh một resource giới hạn, ví dụ CPU, database connection hoặc lock. Tăng concurrency quá cao có thể làm contention tăng và hệ thống chậm hơn thay vì nhanh hơn.

---

# 8. `async/await`

## Bài nói

> `async/await` là cách viết code dựa trên Promise để flow bất đồng bộ dễ đọc hơn. Khi em `await` một Promise đang pending, function hiện tại tạm dừng ở đó; main thread không phải đứng yên chỉ để chờ Promise đó.
>
> Khi Promise hoàn thành, phần code sau `await` được schedule để tiếp tục chạy thông qua Promise/microtask mechanism.
>
> Nhưng `async/await` không tự tạo thread và không biến CPU-heavy synchronous code thành non-blocking.

### Pending và settled Promise là gì?

> Pending nghĩa là Promise chưa có kết quả cuối. Settled nghĩa là nó đã resolve/fulfilled hoặc reject.

### `await` có block thread không?

> Await một Promise thực sự asynchronous không giữ main thread đứng chờ. Nhưng code synchronous chạy trước hoặc sau `await` vẫn chạy trên JavaScript thread và vẫn có thể block.

---

# 9. `Promise.all` và Concurrency

## Bài nói

> `Promise.all` hữu ích khi em có nhiều asynchronous task độc lập. Thay vì await task 1 xong mới bắt đầu task 2, em có thể start nhiều task rồi chờ tất cả hoàn thành.
>
> Nhưng `Promise.all` không tự giới hạn số task đang chạy. Nếu em map 1.000 shop thành 1.000 Promise gọi database/API, em có thể tạo áp lực rất lớn lên connection pool hoặc API rate limit.
>
> Em từng gặp case gần như vậy với batch nhiều shop. Cách xử lý tốt hơn là bounded concurrency, ví dụ chỉ cho 5 hoặc 10 shop xử lý đồng thời, xong một shop mới lấy shop tiếp theo.

### Concurrency là gì?

> Concurrency nghĩa là nhiều task cùng tồn tại trong quá trình xử lý và thời gian của chúng có thể overlap. Với I/O, task A có thể đang chờ database trong lúc application tiến hành task B.

### Parallelism là gì?

> Parallelism nghĩa là nhiều công việc thực sự thực thi cùng một thời điểm, ví dụ trên nhiều CPU core/threads. Concurrency không bắt buộc phải là parallelism.

### Connection pool là gì?

> Là một nhóm database connections được application tái sử dụng. Vì số connection có giới hạn, nếu em phát quá nhiều query cùng lúc thì chúng phải chờ connection rảnh và có thể timeout.

---

# 10. CPU-bound và Worker Threads

> Nếu một request chạy vòng lặp tính toán rất lớn, JavaScript thread bị chiếm cho đến khi đoạn synchronous đó xong. Trong thời gian đó Event Loop không thể cho các JavaScript callbacks khác chạy bình thường.

### Worker Thread là gì?

> Worker Threads cho phép chạy JavaScript trên thread khác trong cùng Node.js process. Nó phù hợp khi mình thực sự có CPU-heavy JavaScript cần offload khỏi main thread.

### Khi nào không cần Worker Thread?

> Nếu task chủ yếu là chờ database/API thì Worker Thread thường không giải quyết đúng vấn đề. Node đã phù hợp với asynchronous I/O. Nếu task dài và không cần response ngay, background job/queue có thể phù hợp hơn.

### Worker Thread vs Child Process?

> Worker Threads là các thread trong cùng process và có thể chia sẻ memory qua cơ chế như `SharedArrayBuffer`. Child Process là process riêng với memory space riêng, isolation mạnh hơn nhưng communication thường tốn overhead hơn.

---

# 11. Streams và Backpressure

## Stream là gì?

> Stream cho phép xử lý dữ liệu từng phần, hay từng chunk, thay vì phải load toàn bộ dữ liệu vào RAM rồi mới xử lý.

**Ví dụ:** file 2 GB. Nếu `readFile` toàn bộ, application có thể cần giữ lượng memory rất lớn. Với stream, em đọc một chunk, xử lý/ghi nó, rồi tiếp tục chunk sau.

### Chunk là gì?

> Chunk đơn giản là một phần nhỏ của luồng dữ liệu, ví dụ một block bytes của file.

### Backpressure là gì?

> Backpressure xảy ra ở bài toán producer-consumer. Producer tạo dữ liệu nhanh hơn consumer xử lý. Nếu producer cứ tiếp tục đẩy vô hạn, buffer/memory có thể tăng.
>
> Cơ chế backpressure cho producer biết cần chậm lại. Với Writable stream, `write()` trả `false` là tín hiệu không nên tiếp tục đẩy dữ liệu ngay; có thể chờ `drain`. `pipe()`/`pipeline()` giúp phối hợp flow này.

### Producer và Consumer là gì?

> Producer là phía tạo/đọc ra dữ liệu; consumer là phía nhận và xử lý/ghi dữ liệu. Ví dụ đọc file là producer và upload destination có thể là consumer.

### Các loại stream?

> Readable để đọc, Writable để ghi, Duplex vừa đọc vừa ghi, Transform vừa nhận dữ liệu vừa biến đổi rồi output dữ liệu khác.

---

# 12. Memory Leak

## Memory leak nghĩa là gì?

> Trong JavaScript có garbage collector, nhưng object chỉ được giải phóng khi không còn reference cần thiết tới nó. Memory leak thường xảy ra khi application vô tình giữ reference tới dữ liệu không còn cần dùng, khiến GC không thể thu hồi.

### Ví dụ

> Cache là một `Map` global và cứ thêm key mới nhưng không TTL/eviction. Dù request đã xong, Map vẫn giữ reference nên memory tiếp tục tăng.

### GC là gì?

> Garbage Collector là cơ chế runtime tự tìm các object không còn reachable để thu hồi memory. Có GC không có nghĩa application không thể leak; nếu code vẫn giữ reference thì GC coi object đó vẫn còn dùng.

### Debug thế nào?

> Em theo dõi heap theo thời gian, kiểm tra xem memory có tăng liên tục sau khi load giảm không, rồi dùng heap snapshot/profiler để xem loại object nào bị giữ lại và reference chain nào giữ chúng.

---

# 13. Graceful Shutdown

## Là gì?

> Graceful shutdown nghĩa là khi process cần dừng, mình không kill ngay giữa lúc đang xử lý request/job. Application ngừng nhận traffic mới, cho các công việc đang chạy một khoảng thời gian để hoàn thành, đóng database/Redis connection, dừng consumer và sau đó exit.

### In-flight request là gì?

> Là request đã vào server và đang được xử lý nhưng chưa trả response xong.

### Tại sao cần timeout cuối?

> Nếu một request hoặc resource treo mãi, process cũng không thể chờ vô hạn. Vì vậy graceful shutdown thường có deadline; quá deadline thì application buộc phải exit và dựa vào retry/idempotency cho công việc cần phục hồi.

---

# 14. Chuỗi câu hỏi interviewer có thể hỏi

### "Node single-thread mà sao xử lý nhiều request?"

> JavaScript callback chủ yếu chạy trên main thread, nhưng phần lớn thời gian I/O là chờ external system. Node dùng asynchronous I/O + Event Loop để không giữ main JS thread đứng chờ từng I/O operation.

### "Async có nghĩa là chạy thread khác không?"

> Không. Async mô tả cách công việc được chờ/schedule. Một số operation có thể dùng libuv worker pool, một số network I/O dùng OS event mechanism, còn Promise bản thân không phải thread.

### "Tại sao CPU-heavy lại block?"

> Vì đoạn JavaScript synchronous CPU-heavy đang chiếm main JS thread. Event Loop không thể chạy JavaScript callback khác cho tới khi đoạn đó nhường thread.

### "Tại sao Stream tiết kiệm memory?"

> Vì xử lý theo chunk thay vì giữ toàn bộ dataset/file trong memory cùng lúc.

### "Backpressure nếu không xử lý thì sao?"

> Producer có thể tiếp tục tạo dữ liệu nhanh hơn consumer giải phóng, buffer tăng và cuối cùng memory/latency có thể tăng mạnh.

## Cách nhớ bằng câu chuyện

`Node chạy JS bằng V8 → JS chủ yếu main thread → I/O không đứng chờ → libuv/OS báo khi sẵn sàng → Event Loop cho callback chạy → Promise có microtask → CPU-heavy block → Worker cho CPU → Stream cho dữ liệu lớn → backpressure khi bên ghi chậm`
