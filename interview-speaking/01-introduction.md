# 01 — Giới thiệu bản thân

Mục tiêu của phần này là **nói tự nhiên, dễ nối sang project thực tế**, không biến phần giới thiệu thành danh sách công nghệ.

---

# 💬 Bài nói 45–60 giây

> Em là Backend Developer, công việc chính của em tập trung vào Node.js và TypeScript. Trong các dự án gần đây em làm khá nhiều với Shopify và các hệ thống tích hợp bên thứ ba.
>
> Công việc của em không chỉ là viết CRUD API mà còn có các bài toán như đồng bộ dữ liệu giữa nhiều hệ thống, xử lý webhook, background job, giới hạn API, database performance và upload file lớn.
>
> Một số vấn đề production em từng gặp là job chạy quá nhiều task cùng lúc làm database connection bị thiếu, dữ liệu local chưa cập nhật kịp theo Shopify, hoặc upload file lớn bị giới hạn ở Nginx/server.
>
> Qua những case đó, em quen với việc trước tiên phải hiểu flow của hệ thống, xác định nguyên nhân gốc rồi mới chọn giải pháp. Em muốn tiếp tục phát triển sâu hơn về backend architecture và system design.

---

# 💬 Bản ngắn 20–30 giây

> Em là Backend Developer, chủ yếu làm Node.js/TypeScript. Em có kinh nghiệm với Shopify integration, PostgreSQL, MongoDB, Redis và AWS. Phần em làm khá nhiều là API integration, data synchronization, background jobs và xử lý các vấn đề production liên quan tới concurrency, database và external API.

---

# 🧠 Tại sao nên giới thiệu như vậy?

Interviewer thường không cần nghe toàn bộ stack ngay từ đầu. Phần giới thiệu nên tạo ra **những nhánh bạn sẵn sàng bị hỏi sâu**.

Ví dụ khi bạn nói:

> “Em từng gặp job chạy quá nhiều task cùng lúc làm database connection bị thiếu.”

Interviewer có thể hỏi:

- Vì sao lại thiếu connection?
- `Promise.all` liên quan gì?
- Connection pool là gì?
- Tại sao không tăng pool?

Đây đều là những phần có case thực tế để kể tiếp.

---

# 🧾 Thuật ngữ trong phần giới thiệu

### **Backend integration** *(kết nối backend của mình với một hệ thống khác)*

Ví dụ application gọi Shopify API, nhận Shopify webhook, gọi Stripe hoặc upload file lên S3.

### **Webhook** *(HTTP request mà hệ thống bên ngoài chủ động gửi cho mình khi có sự kiện)*

📌 Ví dụ: khi inventory thay đổi, Shopify gửi một request tới endpoint của application để báo có thay đổi.

### **Background job** *(công việc chạy ngoài request-response hiện tại)*

📌 Ví dụ: đồng bộ dữ liệu 1.000 shop không cần bắt user chờ trong một HTTP request. Application có thể tạo job và worker xử lý sau.

### **Connection pool** *(một nhóm database connection được giữ sẵn để tái sử dụng)*

Nếu tất cả connection đang bận thì query mới phải chờ. Nếu chờ quá lâu có thể timeout.

### **Concurrency** *(nhiều task cùng đang trong quá trình xử lý)*

Không có nghĩa tất cả JavaScript đều thực sự chạy cùng một lúc trên nhiều CPU core. Với I/O, nhiều task có thể cùng đang chờ database/API.

### **Root cause** *(nguyên nhân gốc)*

Ví dụ Prisma báo timeout là **biểu hiện**. Nguyên nhân gốc có thể là job tạo quá nhiều query cùng lúc khiến connection pool hết connection.

---

# 🎯 Interviewer có thể hỏi tiếp

## Dự án nào em thấy khó nhất?

> Một case em nhớ khá rõ là background job xử lý dữ liệu cho nhiều shop. Ban đầu em dùng `Promise.all` để giảm thời gian chạy. Khi số shop tăng, nhiều database query được start cùng lúc và Prisma bắt đầu timeout khi lấy connection. Em kiểm tra flow của job và connection pool rồi chuyển sang giới hạn số shop được xử lý cùng lúc. Cách này ổn định hơn, đổi lại job chạy lâu hơn một chút.

## Em mạnh nhất phần nào?

> Em thấy mình có kinh nghiệm thực tế nhiều nhất ở backend integration và xử lý flow dữ liệu giữa các hệ thống. Khi làm với Shopify hoặc third-party API, ngoài việc gọi API em thường phải quan tâm tới rate limit, webhook, retry, dữ liệu có thể cập nhật chậm và trường hợp request bị chạy lại.

## Điểm em muốn cải thiện?

> Em muốn đi sâu hơn về system design và cách thiết kế hệ thống khi scale lớn. Trước đây em đã gặp các vấn đề thực tế như connection pool, concurrency, background jobs và external API nên em muốn hệ thống hóa những kinh nghiệm đó thành khả năng thiết kế tốt hơn.

## Tại sao em thích backend?

> Em thích phần backend vì nó không chỉ là viết endpoint. Em thấy thú vị ở việc dữ liệu đi qua nhiều thành phần như API, database, queue, external service rồi mình phải đảm bảo flow vẫn đúng khi có lỗi hoặc tải tăng.

---

# ⚠️ Dễ bị bắt bẻ

## 1. “Em tối ưu performance rất nhiều.”

Câu này quá rộng. Interviewer có thể hỏi ngay “tối ưu gì?” hoặc “đo bằng gì?”.

✅ **Cách nói an toàn:**

> Em từng xử lý một số vấn đề performance cụ thể, ví dụ query/database connection bị quá tải khi background job chạy nhiều task cùng lúc.

## 2. “Em đảm bảo data consistency giữa Shopify và database.”

Nghe như dữ liệu luôn đồng bộ tuyệt đối, trong khi thực tế có thể có độ trễ.

✅ **Cách nói an toàn:**

> Em thiết kế để dữ liệu local được cập nhật qua webhook và có job kiểm tra lại định kỳ để giảm trường hợp bị lệch.

## 3. “Em làm system design.”

Nếu chưa thực sự chịu trách nhiệm thiết kế toàn bộ hệ thống thì không nên nói quá rộng.

✅ **Cách nói an toàn:**

> Em có tham gia các quyết định thiết kế ở mức backend flow, database, job processing và integration; hiện tại em muốn phát triển sâu hơn về system design tổng thể.

---

# 📌 Cách nhớ

Không học thuộc paragraph. Chỉ nhớ 5 điểm:

**Backend → Node.js → Integration → Production problems → Cách phân tích nguyên nhân**

Khi interviewer dừng ở điểm nào thì mở rộng điểm đó.