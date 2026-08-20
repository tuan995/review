# 01 — Giới thiệu bản thân

## Bài nói 45–60 giây

> Em là Backend Developer, công việc chính của em tập trung vào Node.js/TypeScript và các hệ thống tích hợp với bên thứ ba, đặc biệt là Shopify.
>
> Trong các dự án gần đây em làm với NestJS/Express, PostgreSQL, MongoDB, Redis và AWS S3. Ngoài việc viết API, em còn xử lý các luồng như đồng bộ dữ liệu giữa Shopify và hệ thống nội bộ, webhook, background job, giới hạn API và upload file lớn.
>
> Một số vấn đề thực tế em từng gặp là job xử lý quá nhiều shop cùng lúc khiến database hết connection để phục vụ query mới, dữ liệu local cập nhật chậm hơn Shopify, hoặc file lớn upload qua Nginx bị giới hạn kích thước và timeout.
>
> Qua các case đó em quen với cách đi từ biểu hiện của lỗi, kiểm tra flow/log rồi tìm nguyên nhân gốc trước khi sửa. Em cố gắng không chỉ làm cho lỗi biến mất mà còn hiểu vì sao nó xảy ra và cách tránh lặp lại.

## Bản ngắn 20–30 giây

> Em là Backend Developer, chủ yếu làm Node.js/TypeScript và các hệ thống tích hợp Shopify. Em có kinh nghiệm với PostgreSQL, MongoDB, Redis, AWS và background job. Phần em làm khá nhiều là đồng bộ dữ liệu, tích hợp API bên thứ ba và xử lý các vấn đề production như giới hạn API, nhiều task chạy cùng lúc và database connection pool.

---

# Những từ trong phần giới thiệu dễ bị hỏi

### Integration là gì?

> Em dùng từ integration để chỉ việc hệ thống của mình kết nối và trao đổi dữ liệu với hệ thống khác, ví dụ gọi Shopify API, nhận Shopify webhook rồi lưu dữ liệu cần thiết vào database của mình.

### Background job là gì?

> Là công việc không cần hoàn thành ngay trong request của user. Ví dụ đồng bộ dữ liệu của nhiều shop có thể được đưa ra job chạy phía sau để API không phải chờ quá lâu.

### Webhook là gì?

> Webhook là cách hệ thống bên ngoài chủ động gọi vào endpoint của mình khi có sự kiện. Ví dụ khi dữ liệu trên Shopify thay đổi, Shopify có thể gửi webhook để hệ thống của mình cập nhật theo thay vì phải liên tục hỏi Shopify xem có gì thay đổi không.

### Connection pool là gì?

> Application thường giữ một nhóm database connection để tái sử dụng. Khi tất cả connection đang bận, query mới phải chờ. Nếu phải chờ quá lâu thì có thể timeout.

### Root cause là gì?

> Là nguyên nhân gốc tạo ra vấn đề. Ví dụ Prisma báo timeout chỉ là biểu hiện; nguyên nhân gốc có thể là job tạo quá nhiều query cùng lúc làm hết connection pool.

---

# Interviewer có thể hỏi tiếp

### Dự án nào em thấy khó nhất?

> Một case em nhớ khá rõ là job xử lý dữ liệu cho nhiều shop. Ban đầu em dùng `Promise.all` để giảm thời gian xử lý. Khi số lượng shop tăng, số query chạy cùng lúc tăng theo và Prisma bắt đầu timeout khi lấy database connection.
>
> Em kiểm tra lại flow của job và connection pool, sau đó chuyển từ việc cho toàn bộ shop chạy cùng lúc sang giới hạn số shop được xử lý đồng thời. Cách này làm job có thể lâu hơn một chút nhưng database ổn định hơn.

### Em mạnh nhất phần nào?

> Em tự tin nhất ở backend integration và xử lý flow dữ liệu giữa nhiều hệ thống. Khi làm với Shopify hoặc API bên thứ ba, em không chỉ quan tâm gọi API thành công mà còn để ý giới hạn request, retry khi lỗi, dữ liệu cập nhật chậm, webhook gửi trùng và cách khôi phục khi một bước thất bại.

### Điểm em muốn cải thiện?

> Em muốn đi sâu hơn về system design, tức là cách nhìn toàn bộ hệ thống khi số user, dữ liệu hoặc workload tăng lên. Em đã gặp các vấn đề thực tế như connection pool, concurrency và background jobs, nên em muốn hệ thống hóa các kinh nghiệm đó để thiết kế tốt hơn ngay từ đầu.

## Cách nhớ

Không học thuộc paragraph. Chỉ nhớ 5 ý:

`Backend → Node.js → Integration → Vấn đề production → Tìm nguyên nhân rồi mới sửa`
