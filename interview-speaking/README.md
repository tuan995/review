# Backend Interview Speaking Handbook

Tài liệu này dùng để **luyện nói phỏng vấn**, không phải để học thuộc định nghĩa.

## Cách luyện

Với mỗi chủ đề, hãy luyện theo flow:

> Bối cảnh → Vấn đề → Phân tích nguyên nhân → Giải pháp → Vì sao chọn → Trade-off → Kết quả → Câu hỏi đào sâu.

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

- Bắt đầu ngắn, để interviewer hỏi sâu.
- Với câu hỏi kinh nghiệm, ưu tiên ví dụ mình thực sự đã làm.
- Luôn nói được **tại sao**, không chỉ nói đã dùng công nghệ gì.
- Nêu trade-off: giải pháp nào cũng có chi phí.
- Nếu không nhớ chi tiết implementation, nói rõ hướng tiếp cận thay vì đoán.
- Khi mô tả incident: triệu chứng → điều tra → root cause → fix → prevention.

## Công thức 60 giây

> Trong dự án em gặp [problem]. Ban đầu hệ thống [current state]. Khi [scale/event] xảy ra thì [symptom]. Em kiểm tra và xác định nguyên nhân là [root cause]. Em giải quyết bằng [solution]. Em chọn cách này vì [reason]. Trade-off là [trade-off]. Sau đó [result/prevention].

Đây là khung có thể áp dụng cho gần như mọi câu hỏi “Em đã gặp khó khăn gì trong dự án?”.
