# Backend Interview Speaking Handbook

Tài liệu này dùng để **luyện nói phỏng vấn**, không phải để học thuộc định nghĩa.

Mục tiêu của bộ này là:

> **Nói đơn giản trước → thuật ngữ sau → ví dụ thực tế → câu hỏi đào sâu.**

Nếu một ý có thể giải thích bằng tiếng Việt dễ hiểu thì hãy giải thích trước. Sau đó mới gắn tên kỹ thuật để vừa nói tự nhiên, vừa học được thuật ngữ chuyên môn.

---

# Ưu tiên học: không học ngang tất cả các chương

Bộ tài liệu đã khá rộng. Từ giờ **không nên cố học 13 chương với mức độ như nhau**.

## 🟢 Core — phải chắc trước

### 1. Node.js

Ưu tiên: [Node.js & Event Loop](03-nodejs.md)

Các ý cần nói được bằng lời của mình:

- Node.js là gì, phù hợp với I/O-bound workload như thế nào.
- Main JavaScript thread, Event Loop, async I/O.
- `async/await`, Promise, microtask ở mức đủ giải thích.
- `Promise.all` và tại sao có thể gây quá nhiều công việc đồng thời.
- CPU-bound và khi nào cần Worker Threads.
- Stream và backpressure ở mức hiểu hiện tượng.

Không cần bắt đầu bằng libuv internals. Chỉ đi sâu khi interviewer hỏi tiếp.

### 2. NestJS

Repo đã có phần lý thuyết riêng:

- [NestJS core concepts](../nest/core.md)
- [NestJS request lifecycle](../nest/req_lc.md)

Các ý nên ưu tiên:

- **Module** *(gom các thành phần liên quan thành một nhóm chức năng)*.
- **Controller** *(nhận request và chuyển công việc cho service/provider)*.
- **Provider / Service** *(class do Nest quản lý và có thể inject vào nơi khác)*.
- **Dependency Injection — DI** *(Nest tạo và truyền dependency vào class thay vì class tự khởi tạo mọi thứ)*.
- **Middleware** *(xử lý request trước khi tới phần authorization/handler; ví dụ logging, correlation ID)*.
- **Guard** *(quyết định request có được đi tiếp hay không; thường dùng auth/permission)*.
- **Pipe** *(validate hoặc transform input trước khi vào handler)*.
- **Interceptor** *(bọc quanh quá trình chạy handler; có thể log timing, transform response, caching...)*.
- **Exception Filter** *(bắt exception và chuyển thành response có kiểm soát)*.
- Request lifecycle ở mức hiểu flow, không chỉ học thuộc thứ tự.

Cách nhớ đơn giản:

```text
Request
  ↓
Middleware
  ↓
Guard
  ↓
Interceptor (before)
  ↓
Pipe
  ↓
Controller / Handler
  ↓
Service
  ↓
Interceptor (after)
  ↓
Response
```

**Không học thứ tự này như một câu thần chú.** Phải biết mỗi thành phần giải quyết vấn đề gì và vì sao không nhét tất cả logic vào Controller.

### 3. Database

Ưu tiên:

- [Database & Index](04-database-index.md)
- [Transaction & Concurrency](05-transaction-concurrency.md)
- [Promise.all, DB Pool & Concurrency Limit](06-concurrency-limit.md)

Các ý cần chắc:

- Index là gì, tại sao có index vẫn có thể không được dùng.
- Composite index và thứ tự column.
- `EXPLAIN` / execution plan dùng để kiểm tra gì.
- Transaction, commit, rollback, ACID bằng ví dụ.
- Race condition và cách tránh read-check-write bị sai.
- Connection pool là gì và vì sao có thể timeout.
- Concurrency limit và tại sao tăng pool không phải lúc nào cũng là lời giải.

---

## 🟡 Follow-up — học sau khi core ổn

Các phần này rất hữu ích vì nối với kinh nghiệm thực tế, nhưng không nên chiếm thời gian trước Node/Nest/DB:

- [Kinh nghiệm dự án Backend / Shopify](02-project-experience.md)
- [API Integration & Rate Limit](07-api-rate-limit.md)
- [Background Job, Cron & Queue](08-background-jobs.md)
- [Production Problems & Debugging](12-production-problems.md)

Đây là nhóm giúp trả lời câu hỏi kiểu:

> “Em gặp khó khăn gì trong dự án?”

hoặc:

> “Tại sao em chọn cách đó?”

---

## 🔴 Deep dive / Optional — chỉ học khi JD hoặc interviewer dẫn tới

- [Redis & Caching](09-redis-cache.md)
- [AWS S3 & Large File Upload](10-aws-s3.md)
- [Stripe Payment & Webhook](11-stripe-webhook.md)
- [System Design](13-system-design.md)

Không bỏ các chương này, nhưng **không cần học sâu trước khi Node/Nest/DB đã chắc**.

---

# Ba tầng tài liệu

Để tránh bị ngợp, hãy xem repo theo 3 tầng:

## 1. Theory — hiểu kiến thức

Các thư mục cũ như `nodejs/`, `nest/`, `database/`, `postgresql/`, `redis/`...

Dùng khi cần hiểu hoặc tra cứu concept.

## 2. Speaking — biết nói trong interview

Thư mục `interview-speaking/`.

Mục tiêu là biến kiến thức thành câu trả lời tự nhiên, có ví dụ và có nhánh follow-up.

## 3. Rapid Review — chỉ ôn nhanh trước interview

Chưa cần tạo thêm nhiều file ngay. Khi core đã chắc, Rapid Review chỉ nên chứa:

- câu hỏi;
- 3–5 ý chính;
- một ví dụ;
- một câu dễ bị bắt bẻ.

**Không biến Rapid Review thành handbook thứ hai.**

---

# Mức độ sâu trong từng chủ đề

Khi học hoặc bổ sung tài liệu, chia kiến thức thành 3 mức:

### 🟢 Core

Phải tự giải thích được mà không nhìn tài liệu.

Ví dụ:

- Event Loop là gì?
- Dependency Injection giải quyết vấn đề gì?
- Index là gì?
- Transaction để làm gì?

### 🟡 Follow-up

Biết giải thích thêm 2–3 câu nếu interviewer hỏi sâu.

Ví dụ:

- `setImmediate` vs `setTimeout(0)`.
- Guard vs Interceptor.
- Composite index và leftmost prefix.
- Optimistic vs pessimistic locking.

### 🔴 Deep dive

Chỉ đi sâu khi interviewer thật sự hỏi hoặc JD yêu cầu.

Ví dụ:

- libuv internals chi tiết.
- edge case rất sâu của isolation level.
- distributed locking implementation chi tiết.
- sharding strategy ở scale lớn.

**Nguyên tắc:** đừng tự đưa cuộc phỏng vấn vào deep dive nếu câu hỏi ban đầu chưa yêu cầu.

---

# Format của một câu trả lời tốt

Mỗi chủ đề quan trọng nên có một **câu trả lời mặc định 60–90 giây** trước.

Sau đó mới tới phần giải thích sâu.

Format trong các chương:

- 💬 **Bài nói** — đoạn có thể luyện nói trực tiếp.
- 🧠 **Hiểu sâu** — giải thích tại sao.
- 📌 **Ví dụ** — tình huống cụ thể.
- 🎯 **Interviewer hỏi tiếp** — nhánh câu hỏi tiếp theo.
- ⚠️ **Dễ bị bắt bẻ** — câu quá tuyệt đối hoặc quá nhiều jargon.
- ✅ **Cách nói an toàn** — phiên bản rõ và chính xác hơn.
- 🧾 **Thuật ngữ** — từ chuyên môn kèm nghĩa đơn giản.

Ví dụ:

> **Idempotency** *(cùng một thao tác bị chạy lại nhiều lần nhưng không tạo kết quả sai hoặc bị nhân đôi)*.

Mục tiêu không phải học thuộc tiếng Anh mà là **nghe thấy thuật ngữ và hiểu hiện tượng phía sau nó**.

---

# Nguyên tắc dùng thuật ngữ

## 1. Nói hiện tượng trước, gọi tên sau

Ví dụ:

> Khi một cache key hết hạn, có thể hàng trăm request cùng lúc không tìm thấy cache và đều query database. Hiện tượng này thường gọi là **cache stampede**.

Nếu quên tên thuật ngữ, mình vẫn giải thích được vấn đề.

## 2. Nếu nói thuật ngữ, phải tự giải thích được trong 1–2 câu

Ví dụ:

> **Connection pool** *(một nhóm connection database được giữ sẵn để application tái sử dụng; nếu tất cả đang bận thì query mới phải chờ)*.

## 3. Không dùng jargon chỉ để câu trả lời nghe “senior”

❌

> Em optimize throughput bằng bounded concurrency để giảm downstream contention.

✅

> Em giới hạn số shop chạy cùng lúc để database không phải nhận quá nhiều query trong cùng một thời điểm. Cách này làm job ổn định hơn.

Nếu interviewer muốn sâu hơn, lúc đó mới nói **throughput**, **bounded concurrency**, **contention**.

## 4. Tránh câu tuyệt đối

Ví dụ không nói:

> “Có index thì query sẽ nhanh.”

Nên nói:

> “Index có thể giúp database tìm dữ liệu nhanh hơn, nhưng optimizer vẫn có thể chọn scan nếu đọc phần lớn table hoặc index không phù hợp với query.”

---

# Flow trả lời câu hỏi kinh nghiệm

Với câu hỏi **“Em gặp khó khăn gì trong dự án?”**, luyện theo flow:

> **Bối cảnh → Vấn đề → Em kiểm tra gì → Nguyên nhân → Cách xử lý → Tại sao chọn → Đổi lại mất gì → Kết quả → Cách tránh tái diễn.**

Ví dụ:

> Trong một background job em cần xử lý nhiều shop. Ban đầu em dùng `Promise.all` cho toàn bộ danh sách để job chạy nhanh hơn. Khi số shop tăng, số query database chạy cùng lúc tăng mạnh và Prisma bắt đầu timeout khi lấy connection. Em kiểm tra flow của job và connection pool rồi xác định nguyên nhân là quá nhiều task chạy cùng lúc. Em chuyển sang giới hạn số shop xử lý đồng thời. Cách này làm job lâu hơn một chút nhưng database ổn định hơn và giảm timeout.

---

# Một câu chuyện thật có thể dùng cho nhiều câu hỏi

Không cần có 50 câu chuyện khác nhau.

Nên có khoảng **8–12 case thực tế thật sự nhớ rõ**, ví dụ:

- `Promise.all` → connection pool timeout.
- Shopify data sync → webhook + job kiểm tra lại.
- API rate limit.
- Cron chạy nhiều instance.
- File lớn qua Nginx/S3.
- Stripe webhook / trạng thái payment.
- Redis/cache nếu thực tế đã dùng.
- Một incident production mà mình đã tự debug.

Một case tốt có thể trả lời nhiều câu:

- khó khăn đã gặp;
- tối ưu performance;
- concurrency;
- database;
- debugging;
- trade-off;
- nếu làm lại sẽ thay đổi gì.

---

# Luôn chuẩn bị câu “Nếu làm lại bây giờ?”

Sau mỗi case thực tế, tự hỏi:

> **Nếu được làm lại từ đầu, em có thay đổi gì không?**

Câu trả lời không cần phủ nhận cách cũ.

Ví dụ:

> Lúc đó concurrency limit đã giải quyết được timeout. Nếu làm lại với workload lớn hơn, em sẽ thêm metrics rõ hơn cho pool usage và job latency từ đầu, đồng thời dùng queue nếu cần retry/persistence tốt hơn.

Cách trả lời này cho thấy mình hiểu giới hạn của giải pháp, không chỉ nhớ “fix đã làm”.

---

# Nếu không nhớ chi tiết thì nói thế nào?

Không đoán implementation.

Có thể nói:

> Phần configuration cụ thể em không nhớ chính xác con số, nhưng flow em xử lý là...

hoặc:

> Em chưa gặp case đó trực tiếp. Theo cách em hiểu, em sẽ bắt đầu bằng...

Điều quan trọng là phân biệt rõ:

- cái mình **đã làm**;
- cái mình **biết về lý thuyết**;
- cái mình **đang suy luận**.

---

# Không học thuộc nguyên văn

Nếu học thuộc một paragraph dài, interviewer chen ngang rất dễ làm mất mạch.

Thay vào đó mỗi câu trả lời chỉ nhớ 5–7 mốc.

Ví dụ case connection pool:

```text
Job nhiều shop
→ Promise.all
→ query tăng cùng lúc
→ pool hết connection
→ query chờ / timeout
→ giới hạn concurrency
→ DB ổn định hơn
```

Sau đó tự nói thành câu bằng lời của mình.

---

# Quy tắc chống phình tài liệu

Từ giờ **không thêm một chủ đề mới chỉ vì “có thể bị hỏi”**.

Chỉ thêm hoặc mở rộng khi ít nhất một trong các điều sau xảy ra:

1. Nó xuất hiện nhiều trong JD/backend interview.
2. Nó thuộc core Node.js / NestJS / Database.
3. Trong mock interview mình thực sự bị bí ở phần đó.
4. Nó là case mình từng làm và có khả năng cao interviewer hỏi từ CV.

Nếu không thuộc các nhóm trên, để ở mức optional.

---

# Những thuật ngữ hay bị hỏi lại

| Thuật ngữ | Nghĩa đơn giản |
|---|---|
| **Root cause** | Nguyên nhân gốc tạo ra vấn đề, không chỉ biểu hiện bên ngoài |
| **Latency** | Thời gian từ lúc bắt đầu request/task tới lúc có kết quả |
| **Throughput** | Số lượng công việc hoàn thành trong một khoảng thời gian |
| **Source of truth** | Nơi được coi là dữ liệu chính thức khi nhiều hệ thống giữ bản sao |
| **Stale data** | Dữ liệu local chưa kịp cập nhật theo dữ liệu mới nhất |
| **Idempotent** | Chạy lại cùng thao tác nhưng không tạo kết quả sai hoặc nhân đôi |
| **Reconciliation** | Kiểm tra định kỳ và sửa dữ liệu bị lệch giữa hai hệ thống |
| **Backoff** | Retry nhưng chờ một khoảng rồi mới thử lại, thường tăng dần |
| **Connection pool** | Nhóm kết nối database được giữ sẵn để tái sử dụng |
| **Concurrency** | Nhiều task cùng đang trong quá trình xử lý |
| **Parallelism** | Nhiều task thực sự chạy cùng một thời điểm trên nhiều execution resource |
| **Race condition** | Kết quả sai vì nhiều request/task cùng thao tác và thứ tự chạy ảnh hưởng kết quả |
| **Deadlock** | Hai transaction giữ tài nguyên và chờ lẫn nhau nên không bên nào đi tiếp |
| **TTL** | Thời gian một cache/key còn hiệu lực trước khi hết hạn |
| **Backpressure** | Làm phía tạo dữ liệu chậm lại khi phía xử lý không theo kịp |
| **Dependency Injection** | Framework/container tạo dependency và truyền vào class thay vì class tự tạo tất cả dependency |
| **Provider** | Thành phần do Nest quản lý và có thể được inject vào thành phần khác |

---

# Thứ tự học đề xuất

Nếu thời gian hạn chế, học theo thứ tự:

```text
01 Introduction
    ↓
03 Node.js
    ↓
NestJS core + request lifecycle
    ↓
04 Index
    ↓
05 Transaction & Concurrency
    ↓
06 DB Pool / Promise.all
    ↓
02 Project Experience
    ↓
07 API / 08 Jobs / 12 Production
    ↓
09–11 + 13 khi cần
```

Không cần hoàn thành tất cả mới đi phỏng vấn. Mục tiêu là **core chắc trước, breadth sau**.

---

# Cách luyện hiệu quả

1. Đọc `💬 Bài nói` một lần.
2. Đóng tài liệu và tự kể lại bằng lời của mình.
3. Nếu là core, phải tự giải thích được các thuật ngữ chính.
4. Mở phần `🎯 Interviewer hỏi tiếp` và tự trả lời.
5. Với mỗi case thực tế, thêm câu: **“Nếu làm lại em sẽ cải thiện gì?”**
6. Cuối cùng thử nói chủ đề trong 60–90 giây.

Mục tiêu cuối cùng không phải nói giống hệt tài liệu. Mục tiêu là:

> **Hiểu → nói được → bị hỏi sâu vẫn giải thích được → không tự kéo mình vào câu hỏi khó không cần thiết.**
