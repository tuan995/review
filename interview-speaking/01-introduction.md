# 01 — Giới thiệu bản thân

## Bài nói 45–60 giây

> Em là Backend Developer, công việc chính của em tập trung vào Node.js/TypeScript và các hệ thống tích hợp với bên thứ ba. Trong các dự án gần đây em làm nhiều với Shopify, NestJS/Express, PostgreSQL, MongoDB, Redis và AWS S3.
>
> Công việc của em không chỉ là viết API mà còn có các bài toán như đồng bộ dữ liệu giữa nhiều hệ thống, xử lý webhook, background job, API rate limit, database performance và upload file lớn.
>
> Một số vấn đề production em từng phải xử lý là job chạy quá nhiều tác vụ đồng thời làm cạn database connection pool, dữ liệu giữa Shopify và hệ thống nội bộ không đồng bộ ngay lập tức, cũng như upload file lớn gặp giới hạn ở Nginx/server.
>
> Qua các case đó em học được cách nhìn vấn đề từ flow của cả hệ thống, tìm root cause trước rồi mới lựa chọn giải pháp thay vì chỉ fix symptom.

## Bản ngắn 20–30 giây

> Em là Backend Developer, chủ yếu làm Node.js/TypeScript. Em có kinh nghiệm với Shopify integration, PostgreSQL, MongoDB, Redis, AWS và các background job. Phần em làm khá nhiều là API integration, data synchronization và xử lý các vấn đề production như rate limit, concurrency và database connection pool.

## Interviewer có thể hỏi tiếp

### Dự án nào em thấy khó nhất?

> Một case em nhớ khá rõ là job xử lý dữ liệu cho nhiều shop. Ban đầu em dùng `Promise.all` để giảm thời gian xử lý. Khi số lượng shop tăng, số query đồng thời tăng theo và Prisma bắt đầu timeout khi lấy connection. Sau khi kiểm tra connection pool và flow của job, em chuyển sang giới hạn concurrency thay vì cho toàn bộ shop chạy cùng lúc.

### Em mạnh nhất phần nào?

> Em tự tin nhất ở backend integration và xử lý flow dữ liệu giữa nhiều hệ thống. Khi làm với Shopify hoặc các third-party API, ngoài gọi API em thường phải quan tâm rate limit, retry, webhook, idempotency, đồng bộ dữ liệu và failure recovery.

### Điểm em muốn cải thiện?

> Em muốn đi sâu hơn về system design và cách thiết kế hệ thống khi scale lớn. Trước đây em đã gặp các vấn đề thực tế như connection pool, concurrency và background jobs, nên em muốn hệ thống hóa các kinh nghiệm đó thành kiến thức thiết kế tốt hơn.

## Cách nhớ

Không học thuộc paragraph. Chỉ nhớ 5 điểm:

`Backend → Node.js → Integration → Production problems → Root cause thinking`
