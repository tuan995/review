# Backend Interview Speaking Handbook

Tài liệu này dùng để **luyện nói phỏng vấn**, không phải để học thuộc định nghĩa.

Mục tiêu là: **nói đơn giản trước, thuật ngữ sau**. Nếu một ý có thể giải thích bằng câu tiếng Việt dễ hiểu thì ưu tiên giải thích trước. Chỉ dùng thuật ngữ kỹ thuật khi nó thực sự giúp câu trả lời chính xác hơn.

## Cách luyện

Với mỗi chủ đề, hãy luyện theo flow:

> Bối cảnh → Vấn đề → Em kiểm tra gì → Nguyên nhân → Cách xử lý → Tại sao chọn → Đổi lại mất gì → Kết quả → Câu hỏi đào sâu.

Không cần học thuộc từng chữ. Hãy nhớ **ý chính và câu chuyện**.

## Nội dung

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

## Nguyên tắc khi trả lời

- **Nói ý đơn giản trước.** Ví dụ thay vì mở đầu bằng “eventual consistency”, hãy nói “dữ liệu local có thể chậm hơn Shopify một khoảng ngắn”, rồi nếu cần mới gọi đó là eventual consistency.
- Nếu dùng một từ dễ bị hỏi như `idempotency`, `backpressure`, `connection pool`, `reconciliation`, `deadlock`..., phải tự giải thích được bằng **1–2 câu và một ví dụ**.
- Không dùng thuật ngữ chỉ để câu trả lời nghe “senior” hơn. Thuật ngữ nào không cần thiết thì bỏ.
- Bắt đầu ngắn, để interviewer hỏi sâu. Nhưng câu đầu phải đủ nghĩa, không phải một chuỗi keyword.
- Với câu hỏi kinh nghiệm, ưu tiên ví dụ mình thực sự đã làm.
- Luôn nói được **tại sao**, không chỉ nói đã dùng công nghệ gì.
- Khi nói `trade-off`, có thể nói tự nhiên hơn là **“đổi lại…”**. Ví dụ: “Cách này ổn định hơn, đổi lại job chạy lâu hơn một chút.”
- Nếu không nhớ chi tiết implementation, nói rõ hướng tiếp cận thay vì đoán.
- Khi mô tả sự cố: **biểu hiện → ảnh hưởng → kiểm tra → nguyên nhân → sửa → kiểm tra lại → ngăn tái diễn**.

## Những từ dễ bị hỏi mẹo

Không cần tránh hoàn toàn, nhưng khi nói ra phải biết nghĩa đơn giản:

| Thuật ngữ | Cách nói đơn giản |
|---|---|
| Root cause | Nguyên nhân gốc gây ra vấn đề, không chỉ biểu hiện bên ngoài |
| Latency | Thời gian từ lúc bắt đầu request/task đến lúc nhận kết quả |
| Throughput | Số lượng công việc xử lý được trong một khoảng thời gian |
| Source of truth | Nơi được coi là dữ liệu chính thức khi hai hệ thống khác nhau |
| Stale data | Dữ liệu local chưa kịp cập nhật theo nguồn mới nhất |
| Idempotent | Chạy lại cùng một thao tác nhưng không tạo kết quả sai hoặc bị nhân đôi |
| Reconciliation | Job định kỳ kiểm tra và sửa dữ liệu bị lệch giữa hai hệ thống |
| Backoff | Khi retry thì chờ một khoảng rồi mới thử lại, thường tăng dần |
| Jitter | Thêm một khoảng ngẫu nhiên nhỏ vào thời gian retry để nhiều worker không retry cùng lúc |
| Connection pool | Một nhóm kết nối DB được tái sử dụng; hết connection thì query mới phải chờ |
| Concurrency | Nhiều task cùng đang trong quá trình xử lý |
| Race condition | Kết quả bị sai do nhiều request/task cùng sửa dữ liệu và thứ tự chạy ảnh hưởng kết quả |
| Deadlock | Hai transaction giữ resource và chờ lẫn nhau nên không bên nào đi tiếp được |
| TTL | Thời gian một cache/key còn hiệu lực trước khi hết hạn |

## Công thức 60–90 giây

> Trong dự án em gặp [vấn đề]. Ban đầu hệ thống đang [cách làm hiện tại]. Khi [số lượng/traffic/tình huống] tăng thì em thấy [biểu hiện]. Em kiểm tra [log/metrics/flow] và xác định nguyên nhân là [nguyên nhân]. Em xử lý bằng [giải pháp]. Em chọn cách này vì [lý do]. Đổi lại [chi phí/hạn chế]. Sau đó em kiểm tra lại bằng [cách verify] và bổ sung [cách ngăn tái diễn].

Đây là khung có thể áp dụng cho gần như mọi câu hỏi “Em đã gặp khó khăn gì trong dự án?”.
