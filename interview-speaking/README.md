# Backend Interview Speaking Handbook

Tài liệu này dùng để **luyện nói phỏng vấn**, không phải để học thuộc định nghĩa.

Mục tiêu của bộ này là:

> **Nói đơn giản trước → thuật ngữ sau → ví dụ thực tế → câu hỏi đào sâu.**

Nếu một ý có thể giải thích bằng tiếng Việt dễ hiểu thì hãy giải thích trước. Sau đó mới gắn tên kỹ thuật để vừa nói tự nhiên, vừa học được thuật ngữ chuyên môn.

---

# Cách đọc tài liệu

Mỗi chương cố gắng dùng cùng một format:

- 💬 **Bài nói** — đoạn có thể luyện nói trực tiếp trong interview.
- 🧠 **Hiểu sâu** — giải thích tại sao cơ chế đó hoạt động như vậy.
- 📌 **Ví dụ** — tình huống cụ thể để dễ nhớ.
- 🎯 **Interviewer hỏi tiếp** — câu hỏi có thể bị đào sâu.
- ⚠️ **Dễ bị bắt bẻ** — cách nói quá ngắn hoặc dễ tạo câu hỏi khó.
- ✅ **Cách nói an toàn** — phiên bản rõ hơn, chính xác hơn.
- 🧾 **Thuật ngữ** — từ chuyên môn kèm nghĩa đơn giản.

Ví dụ:

> **Idempotency** *(cùng một thao tác bị chạy lại nhiều lần nhưng không tạo kết quả sai hoặc bị nhân đôi)*.

Khi thấy thuật ngữ được viết như trên, mục tiêu không phải học thuộc tiếng Anh mà là **hiểu được nghĩa bằng một câu đơn giản**.

---

# Flow trả lời câu hỏi kinh nghiệm

Với câu hỏi kiểu **“Em gặp khó khăn gì trong dự án?”**, luyện theo flow:

> **Bối cảnh → Vấn đề → Em kiểm tra gì → Nguyên nhân → Cách xử lý → Tại sao chọn → Đổi lại mất gì → Kết quả → Cách tránh tái diễn.**

Ví dụ:

> Trong một background job em cần xử lý nhiều shop. Ban đầu em dùng `Promise.all` cho toàn bộ danh sách để job chạy nhanh hơn. Khi số shop tăng, số query database chạy cùng lúc tăng rất mạnh và Prisma bắt đầu timeout khi lấy connection. Em kiểm tra flow của job và connection pool rồi xác định nguyên nhân là số task đồng thời quá lớn. Em chuyển sang giới hạn số shop xử lý cùng lúc. Cách này làm job chậm hơn một chút nhưng database ổn định hơn và giảm timeout.

Đây là kiểu trả lời tốt hơn việc chỉ nói:

> “Em gặp connection pool issue và fix bằng concurrency limit.”

Vì câu thứ hai có quá nhiều thuật ngữ nhưng không cho interviewer thấy **em hiểu nguyên nhân và cách suy nghĩ**.

---

# Nguyên tắc dùng thuật ngữ

## 1. Không tránh thuật ngữ hoàn toàn

Biết thuật ngữ là tốt. Nhưng khi nói ra phải tự giải thích được.

Ví dụ:

> Em dùng **reconciliation job** *(job định kỳ kiểm tra lại hai hệ thống và sửa dữ liệu bị lệch)*.

Thay vì:

> Em dùng reconciliation để đảm bảo eventual consistency.

Câu thứ hai đúng về mặt ý tưởng nhưng chứa hai điểm interviewer có thể hỏi ngược ngay.

## 2. Nói hiện tượng trước, gọi tên sau

Ví dụ với **cache stampede**:

> Khi một cache key hết hạn, có thể hàng trăm request cùng lúc không tìm thấy cache và đều query database. Hiện tượng này thường gọi là **cache stampede**.

Cách này giúp dù quên tên thuật ngữ, bạn vẫn giải thích được vấn đề.

## 3. Không dùng thuật ngữ để “nghe senior”

Nếu từ đó không làm câu trả lời rõ hơn thì bỏ.

Ví dụ:

❌ “Em optimize throughput bằng bounded concurrency để giảm downstream contention.”

✅ “Em giới hạn số shop chạy cùng lúc để database không phải nhận quá nhiều query trong cùng một thời điểm. Nhờ vậy job ổn định hơn.”

Sau đó nếu interviewer hỏi sâu mới dùng các từ **throughput**, **bounded concurrency**, **contention**.

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
| **Jitter** | Thêm một khoảng ngẫu nhiên nhỏ vào thời gian retry để các worker không retry cùng lúc |
| **Connection pool** | Nhóm kết nối database được giữ sẵn để tái sử dụng |
| **Concurrency** | Nhiều task cùng đang trong quá trình xử lý |
| **Parallelism** | Nhiều task thực sự chạy cùng một thời điểm trên nhiều execution resource |
| **Race condition** | Kết quả sai vì nhiều request/task cùng thao tác và thứ tự chạy ảnh hưởng kết quả |
| **Deadlock** | Hai transaction giữ tài nguyên và chờ lẫn nhau nên không bên nào đi tiếp |
| **TTL** | Thời gian một cache/key còn hiệu lực trước khi hết hạn |
| **Backpressure** | Làm phía tạo dữ liệu chậm lại khi phía xử lý không theo kịp |
| **Stateless** | Một API instance không phụ thuộc vào dữ liệu session chỉ nằm trong memory của chính instance đó |

---

# Công thức 60–90 giây

> Trong dự án em gặp **[vấn đề]**. Ban đầu hệ thống đang **[cách làm]**. Khi **[scale/tình huống]** xảy ra thì em thấy **[biểu hiện]**. Em kiểm tra **[log/metrics/flow]** và xác định nguyên nhân là **[nguyên nhân]**. Em xử lý bằng **[giải pháp]** vì **[lý do]**. Đổi lại **[chi phí/hạn chế]**. Sau đó em kiểm tra lại bằng **[cách verify]** và bổ sung **[cách tránh tái diễn]**.

Không cần học thuộc nguyên đoạn. Chỉ cần nhớ flow.

---

# Nội dung

1. [Giới thiệu bản thân](01-introduction.md)
2. [Kinh nghiệm dự án Backend / Shopify](02-project-experience.md)
3. [Node.js & Event Loop](03-nodejs.md)
4. [Database & Index](04-database-index.md)
5. [Transaction & Concurrency](05-transaction-concurrency.md)
6. [Promise.all, DB Pool & Concurrency Limit](06-concurrency-limit.md)
7. [API Integration & Rate Limit](07-api-rate-limit.md)
8. [Background Job, Cron & Queue](08-background-jobs.md)
9. [Redis & Caching](09-redis-cache.md)
10. [AWS S3 & Large File Upload](10-aws-s3.md)
11. [Stripe Payment & Webhook](11-stripe-webhook.md)
12. [Production Problems & Debugging](12-production-problems.md)
13. [System Design Speaking Framework](13-system-design.md)

---

# Cách luyện hiệu quả

1. Đọc `💬 Bài nói` một lần.
2. Đóng tài liệu và tự kể lại bằng lời của mình.
3. Mở phần `🎯 Interviewer hỏi tiếp` và tự trả lời.
4. Gặp thuật ngữ nào chưa giải thích được thì quay lại `🧾 Thuật ngữ`.
5. Cuối cùng thử nói lại toàn bộ chủ đề trong 60–90 giây.

Mục tiêu cuối cùng không phải nói giống hệt tài liệu. Mục tiêu là **hiểu đủ để dùng từ của chính mình**.