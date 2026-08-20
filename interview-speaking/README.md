# Backend Interview Speaking Handbook

> **Mục tiêu:** biến kiến thức Backend thành câu trả lời phỏng vấn tự nhiên, dễ nhớ, có ví dụ thực tế và vẫn trả lời được khi interviewer hỏi sâu.

Bộ này **không phải một giáo trình để học thuộc**. Quy tắc chung là:

> **Nói đơn giản trước → thuật ngữ sau → ví dụ thực tế → câu hỏi đào sâu.**

Nếu một ý có thể giải thích bằng tiếng Việt dễ hiểu thì giải thích trước. Sau đó mới gắn tên kỹ thuật để vừa nói tự nhiên, vừa biết đúng thuật ngữ.

---

# 1. GitHub là nguồn chuẩn duy nhất

Repo này là **single source of truth** *(nguồn chính thức duy nhất của handbook)*.

Quy tắc:

- Nội dung đã chốt phải được đưa về GitHub.
- Chat chỉ dùng để **luyện, sửa, bổ sung hoặc phát hiện điểm yếu**.
- Không coi một đoạn trả lời nằm rải rác trong chat là bản chính thức nếu chưa được cập nhật vào repo.
- Có thể mở nhiều chat mock interview khác nhau mà không sợ tài liệu bị phân tán, vì bản chuẩn vẫn ở đây.

Nếu một chat luyện phát hiện phần nào còn yếu, quay lại chat quản lý handbook và cập nhật đúng phần đó vào GitHub.

---

# 2. Không học ngang tất cả các chương

Bộ tài liệu đã đủ rộng. Từ giờ **không mở rộng lan man** và không cố học mọi thứ với cùng độ sâu.

## 🟢 Core — phải chắc trước

### Node.js

Ưu tiên: [Node.js & Event Loop](03-nodejs.md)

Cần nói được:

- Node.js là gì và phù hợp với I/O-bound workload như thế nào.
- Main JavaScript thread, Event Loop, async I/O.
- Promise, `async/await`, microtask ở mức đủ giải thích.
- `Promise.all` và vấn đề quá nhiều task đồng thời.
- CPU-bound và khi nào cần Worker Threads.
- Stream và backpressure ở mức hiểu hiện tượng.

Không tự đi sâu vào libuv internals nếu interviewer chưa hỏi.

### NestJS

Theory đã có ở:

- [NestJS core concepts](../nest/core.md)
- [NestJS request lifecycle](../nest/req_lc.md)

Cần chắc:

- **Module** *(gom các thành phần liên quan thành một nhóm chức năng)*.
- **Controller** *(nhận request và chuyển xử lý cho service/provider)*.
- **Provider / Service** *(class do Nest quản lý và có thể inject vào nơi khác)*.
- **Dependency Injection — DI** *(Nest tạo và truyền dependency vào class thay vì class tự tạo tất cả dependency)*.
- **Middleware** *(xử lý request sớm, ví dụ logging/correlation ID)*.
- **Guard** *(quyết định request có được đi tiếp hay không; thường dùng auth/permission)*.
- **Pipe** *(validate hoặc transform input trước khi vào handler)*.
- **Interceptor** *(bọc quanh handler; có thể log timing, transform response, cache...)*.
- **Exception Filter** *(bắt exception và chuyển thành response có kiểm soát)*.
- Request lifecycle ở mức hiểu **mỗi thành phần giải quyết vấn đề gì**, không chỉ thuộc thứ tự.

Mental model:

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

### Database

Ưu tiên:

- [Database & Index](04-database-index.md)
- [Transaction & Concurrency](05-transaction-concurrency.md)
- [Promise.all, DB Pool & Concurrency Limit](06-concurrency-limit.md)

Cần chắc:

- Index là gì, khi nào database có thể không dùng index.
- Composite index và thứ tự column.
- `EXPLAIN` / execution plan để kiểm tra gì.
- Transaction, commit, rollback, ACID bằng ví dụ.
- Race condition và vấn đề read-check-write.
- Connection pool là gì và tại sao có thể timeout.
- Concurrency limit và tại sao tăng pool không phải lúc nào cũng là lời giải.

---

## 🟡 Follow-up — học sau khi core ổn

- [Project Experience](02-project-experience.md)
- [API Integration & Rate Limit](07-api-rate-limit.md)
- [Background Job, Cron & Queue](08-background-jobs.md)
- [Production Problems & Debugging](12-production-problems.md)

Đây là nhóm dùng nhiều cho các câu:

- “Em gặp khó khăn gì trong dự án?”
- “Tại sao em chọn cách đó?”
- “Em debug production issue như thế nào?”

---

## 🔴 Deep dive / Optional

- [Redis & Caching](09-redis-cache.md)
- [AWS S3 & Large File Upload](10-aws-s3.md)
- [Stripe Payment & Webhook](11-stripe-webhook.md)
- [System Design](13-system-design.md)

Giữ để dùng khi JD hoặc interviewer dẫn tới. **Không học sâu trước khi Node/Nest/DB đã chắc.**

---

# 3. Ba tầng tài liệu

## Theory — để hiểu

Các thư mục cũ như `nodejs/`, `nest/`, `database/`, `postgresql/`, `redis/`...

Dùng để học bản chất hoặc tra cứu.

## Speaking — để nói

Thư mục `interview-speaking/`.

Mỗi chủ đề nên có:

- 💬 **Bài nói** — câu trả lời có thể luyện trực tiếp.
- 🧠 **Hiểu sâu** — giải thích cơ chế.
- 📌 **Ví dụ** — case cụ thể.
- 🎯 **Interviewer hỏi tiếp** — nhánh đào sâu.
- ⚠️ **Dễ bị bắt bẻ** — câu quá tuyệt đối hoặc quá nhiều jargon.
- ✅ **Cách nói an toàn** — phiên bản rõ và chính xác hơn.
- 🧾 **Thuật ngữ** — từ chuyên môn + nghĩa đơn giản.

## Rapid Review — chỉ ôn nhanh

Chỉ làm khi core đã chắc.

Mỗi câu chỉ cần:

- câu hỏi;
- 3–5 ý chính;
- một ví dụ;
- một bẫy dễ nói sai.

**Không biến Rapid Review thành handbook thứ hai.**

---

# 4. Xây 8–12 “câu chuyện thật” từ project

Đây là phần rất quan trọng.

Interviewer thường nhớ một case thực tế rõ ràng hơn một định nghĩa hoàn hảo. Không cần 50 case. Chỉ cần khoảng **8–12 câu chuyện thật mình nhớ rõ** và dùng lại chúng cho nhiều loại câu hỏi.

Các story nên ưu tiên:

1. `Promise.all` → nhiều query → connection pool timeout.
2. Shopify data sync → local DB thay vì gọi API liên tục.
3. Inventory thay đổi → webhook + job kiểm tra lại.
4. API rate limit / 429 / retry.
5. Retry request tạo dữ liệu → nguy cơ duplicate / idempotency.
6. Upload file lớn ~170MB → Nginx limit / S3 direct upload.
7. Cron chạy duplicate khi có nhiều process/instance.
8. Redis/cache — chỉ dùng case thực tế mình đã làm và nhớ rõ.
9. Stripe subscription/webhook/state sync.
10. Một production incident khác mình trực tiếp debug và có thể kể rõ.

Một story tốt có thể dùng cho nhiều câu hỏi:

- khó khăn trong dự án;
- Node.js concurrency;
- database;
- performance;
- debugging;
- architecture;
- trade-off;
- failure handling;
- “nếu làm lại sẽ thay đổi gì?”.

**Không bịa thêm case chỉ để đủ số lượng.** Story nào không nhớ rõ thì không dùng như kinh nghiệm thật.

---

# 5. Cấu trúc bắt buộc của một Project Story

Mỗi story nên đi theo flow:

> **Bối cảnh → Vấn đề → Em kiểm tra gì → Nguyên nhân → Giải pháp → Tại sao chọn → Đổi lại mất gì → Kết quả → Nếu làm lại → Cách tránh tái diễn.**

Ví dụ connection pool:

> Trong một background job em cần xử lý nhiều shop. Ban đầu em dùng `Promise.all` cho toàn bộ danh sách để job chạy nhanh hơn. Khi số shop tăng, số query database chạy cùng lúc tăng mạnh và Prisma bắt đầu timeout khi lấy connection. Em kiểm tra flow của job và connection pool rồi xác định nguyên nhân là quá nhiều task cùng dùng database. Em chuyển sang giới hạn số shop xử lý đồng thời. Cách này làm job lâu hơn một chút nhưng database ổn định hơn và giảm timeout.

## Phải có câu “Nếu làm lại bây giờ?”

Sau mỗi story, tự hỏi:

> **Nếu được làm lại từ đầu, em sẽ cải thiện gì?**

Ví dụ:

> Lúc đó concurrency limit đã giải quyết được timeout. Nếu làm lại với workload lớn hơn, em sẽ thêm metrics cho pool usage và job latency ngay từ đầu, đồng thời cân nhắc queue nếu cần retry và persistence tốt hơn.

Mục tiêu không phải chê cách cũ. Mục tiêu là cho thấy mình hiểu **giới hạn của giải pháp và điều mình học được sau đó**.

---

# 6. Không học thuộc nguyên văn — chỉ nhớ 5–7 mốc

Không học một paragraph dài.

Interviewer chen ngang một câu là rất dễ mất mạch nếu mình đang đọc thuộc trong đầu.

Mỗi câu trả lời chỉ nhớ 5–7 mốc.

Ví dụ:

```text
Nhiều shop
→ Promise.all
→ query tăng cùng lúc
→ pool hết connection
→ query chờ / timeout
→ limit concurrency
→ DB ổn định hơn
```

Từ các mốc đó, tự nói lại bằng lời của mình.

Nếu cách nói mỗi lần hơi khác nhau nhưng ý vẫn đúng thì **đó là tín hiệu tốt** — nghĩa là mình hiểu chứ không chỉ thuộc.

---

# 7. Khi không chắc thì nói thế nào?

**Không đoán để lấp chỗ trống.**

Phải phân biệt rõ:

- cái mình **đã trực tiếp làm**;
- cái mình **biết về lý thuyết**;
- cái mình **đang suy luận / đề xuất nếu gặp lại**.

Có thể dùng các câu sau:

> **“Phần implementation/config cụ thể em không nhớ chính xác, nhưng flow em xử lý lúc đó là…”**

> **“Em không nhớ chính xác con số nên em không muốn khẳng định, nhưng nguyên tắc em dùng là…”**

> **“Phần này em chưa trực tiếp triển khai sâu. Theo cách em hiểu thì…”**

> **“Case đó em chưa gặp trực tiếp. Nếu gặp, em sẽ bắt đầu kiểm tra từ…”**

> **“Em nhớ hướng xử lý, còn API/config chi tiết em sẽ cần kiểm tra lại documentation để chắc chắn.”**

Đây không phải né câu hỏi. Đây là cách **không biến một chi tiết mình không nhớ thành một câu trả lời sai**.

---

# 8. Nguyên tắc dùng thuật ngữ

## Nói hiện tượng trước, gọi tên sau

Ví dụ:

> Khi một cache key hết hạn, hàng trăm request có thể cùng miss cache và cùng query database. Hiện tượng này thường gọi là **cache stampede**.

## Nếu dùng thuật ngữ, phải giải thích được trong 1–2 câu

Ví dụ:

> **Connection pool** *(một nhóm connection database được giữ sẵn để application tái sử dụng; nếu tất cả đang bận thì query mới phải chờ)*.

## Không dùng jargon chỉ để nghe “senior”

❌

> Em optimize throughput bằng bounded concurrency để giảm downstream contention.

✅

> Em giới hạn số shop chạy cùng lúc để database không nhận quá nhiều query trong cùng một thời điểm. Cách này làm job ổn định hơn.

Nếu interviewer hỏi sâu, lúc đó mới dùng `throughput`, `bounded concurrency`, `contention`.

## Tránh câu tuyệt đối

❌ “Có index thì query sẽ nhanh.”

✅ “Index có thể giúp database tìm dữ liệu nhanh hơn, nhưng optimizer vẫn có thể chọn scan nếu index không phù hợp hoặc query phải đọc phần lớn table.”

---

# 9. Mức độ sâu khi trả lời

### 🟢 Core

Phải tự giải thích được mà không nhìn tài liệu.

### 🟡 Follow-up

Biết thêm 2–3 câu nếu interviewer đào sâu.

### 🔴 Deep dive

Chỉ đi sâu khi interviewer thật sự yêu cầu.

**Không tự kéo mình vào deep dive bằng cách ném ra quá nhiều thuật ngữ trong câu mở đầu.**

---

# 10. English speaking — làm sau, không tạo một bộ kiến thức mới

Sau khi bản Việt đã chắc, có thể tạo bản English speaking.

Quy tắc:

- Nội dung kỹ thuật phải **giống bản Việt**.
- Cùng story, cùng flow, cùng trade-off, cùng follow-up.
- Chỉ thay cách diễn đạt ngôn ngữ.
- Không tạo thêm kiến thức mới chỉ vì đang viết tiếng Anh.

Như vậy mình chỉ học **một hệ thống kiến thức**, không phải hai handbook khác nhau.

---

# 11. Quy tắc chống phình tài liệu

Từ giờ **không thêm chủ đề chỉ vì “có thể interviewer hỏi”**.

Chỉ thêm hoặc mở rộng khi ít nhất một điều đúng:

1. Nó xuất hiện thường xuyên trong Backend JD/interview.
2. Nó thuộc core Node.js / NestJS / Database.
3. Khi mock interview mình thực sự bị bí ở đó.
4. Nó nằm trong CV/project và có khả năng cao bị hỏi.
5. Nó giúp hoàn thiện một trong 8–12 project stories thật.

Nếu không thuộc các nhóm trên, để optional.

---

# 12. Cách dùng ChatGPT mà không làm loãng tài liệu

Nên tách chat theo vai trò.

## Chat quản lý handbook

Dùng để:

- sửa Markdown;
- cập nhật GitHub;
- thêm/sửa project story;
- bổ sung câu hỏi thực tế sau khi luyện;
- kiểm tra handbook có bị phình hoặc trùng không.

**Mẫu mở chat:**

> “Đây là chat quản lý Backend Interview Handbook của tôi. GitHub `tuan995/review` là source of truth. Hãy ưu tiên Node.js, NestJS và Database; không mở rộng lan man. Khi sửa tài liệu, giữ format: bài nói → hiểu sâu → thuật ngữ → ví dụ → follow-up → dễ bị bắt bẻ → cách nói an toàn. Với project story phải có ‘Nếu làm lại bây giờ?’ và 5–7 mốc để nhớ.”

## Chat mock interview

Dùng để luyện nói, không chỉnh handbook ngay trong lúc luyện.

**Mẫu mở chat:**

> “Mock interview Backend cho tôi, ưu tiên Node.js → NestJS → Database và project experience. Hỏi từng câu một. Nếu tôi dùng thuật ngữ, hãy hỏi sâu vào đúng thuật ngữ đó. Nếu câu trả lời của tôi có chỗ mơ hồ hoặc dễ bị bắt bẻ, chỉ ra sau khi tôi trả lời. Không hỏi quá nhiều chủ đề optional nếu tôi chưa chắc core.”

## Sau buổi mock interview

Cuối buổi có thể yêu cầu:

> “Tổng hợp những chỗ tôi bị bí thành danh sách ngắn: câu hỏi → tôi thiếu gì → nên cập nhật file nào trong handbook. Không viết lại toàn bộ handbook.”

Sau đó quay về chat quản lý handbook để cập nhật GitHub.

---

# 13. Những thuật ngữ hay bị hỏi lại

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
| **Parallelism** | Nhiều task thực sự thực thi cùng một thời điểm |
| **Race condition** | Kết quả bị sai vì thứ tự nhiều request/task cùng thao tác ảnh hưởng kết quả |
| **Deadlock** | Hai transaction giữ resource và chờ lẫn nhau nên không bên nào đi tiếp |
| **TTL** | Thời gian cache/key còn hiệu lực trước khi hết hạn |
| **Backpressure** | Làm phía tạo dữ liệu chậm lại khi phía xử lý không theo kịp |
| **Dependency Injection** | Framework/container tạo dependency và truyền vào class thay vì class tự tạo tất cả |
| **Provider** | Thành phần do Nest quản lý và có thể inject vào thành phần khác |

---

# 14. Thứ tự học đề xuất

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
02 Project Experience + 8–12 stories
    ↓
07 API / 08 Jobs / 12 Production
    ↓
09–11 + 13 khi thật sự cần
```

Không cần hoàn thành tất cả mới đi phỏng vấn.

> **Core chắc trước → project story thật → breadth sau.**

---

# 15. Cách luyện một chủ đề

1. Đọc `💬 Bài nói` một lần.
2. Đóng tài liệu.
3. Nhớ 5–7 mốc và tự kể lại.
4. Tự giải thích các thuật ngữ mình vừa dùng.
5. Mở `🎯 Interviewer hỏi tiếp` và trả lời.
6. Nếu là project story, trả lời thêm: **“Nếu làm lại bây giờ?”**
7. Nếu không nhớ chi tiết, dùng cách nói an toàn thay vì đoán.
8. Sau mock interview, chỉ cập nhật những chỗ mình thực sự bị bí.

Mục tiêu cuối cùng:

> **Hiểu → nói được → bị hỏi sâu vẫn giải thích được → không tự kéo mình vào câu hỏi khó không cần thiết.**
