# Node.js Interview Notes

## Node.js là gì?
Node.js là JavaScript runtime chạy trên V8, mạnh với I/O-bound workload nhờ event-driven, non-blocking I/O và event loop.

## Event Loop
JavaScript callback chủ yếu chạy trên main thread. I/O có thể được OS/libuv xử lý bất đồng bộ rồi callback quay lại queue.

## CPU-bound
Code CPU-heavy chạy lâu có thể block event loop. Hướng xử lý: Worker Threads, background worker, queue hoặc tách service.

## async/await
`async/await` là syntax dựa trên Promise. Nó không tự tạo thread mới và không biến CPU-heavy work thành non-blocking.

## Promise.all
Dùng cho task độc lập có thể chạy đồng thời. Cần giới hạn concurrency để tránh overload DB/API.

## Streams
Streams xử lý dữ liệu theo chunk, giảm memory footprint. Các loại chính: Readable, Writable, Duplex, Transform.

## Backpressure
Khi producer nhanh hơn consumer, backpressure giúp tránh memory tăng không kiểm soát.

## Worker Threads
Phù hợp cho CPU-bound task như parsing nặng, image processing hoặc computation.

## Memory leak thường gặp
- Cache không giới hạn.
- Event listener không cleanup.
- Timer giữ reference.
- Global collection tăng mãi.
- Closure giữ object lớn.

## Graceful shutdown
Ngừng nhận traffic mới, chờ request đang chạy, đóng DB/Redis connections, flush telemetry cần thiết rồi exit.

## Câu hỏi phỏng vấn
### Node.js single-threaded nghĩa là gì?
JavaScript execution chủ yếu trên một main thread, nhưng runtime vẫn có async I/O và thread pool cho một số tác vụ.

### Khi nào Node.js không phù hợp?
CPU-bound workload nặng nếu xử lý trực tiếp trên event loop mà không offload.

### async/await có tạo thread mới không?
Không.

### Stream có lợi gì?
Xử lý dữ liệu lớn theo chunk và giảm memory footprint.

## Cheat sheet
- Node mạnh với I/O-bound.
- CPU-heavy có thể block event loop.
- async/await không đồng nghĩa multi-thread.
- Promise.all cần kiểm soát concurrency.
- Stream cho dữ liệu lớn.
- Worker Threads cho CPU-bound.
