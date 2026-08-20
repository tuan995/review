# 12 — Production Problems & Debugging

# Cách trả lời câu “Em từng gặp vấn đề production nào?”

Khi được hỏi, em không bắt đầu ngay bằng solution. Em đi theo flow:

> Biểu hiện → Ảnh hưởng → Em kiểm tra gì → Nguyên nhân → Cách sửa → Kiểm tra lại → Cách tránh lặp lại

Nếu interviewer dùng từ `incident`, em hiểu đơn giản là **một sự cố thực tế xảy ra trên môi trường đang phục vụ user/job thật**.

---

# Case 1 — Prisma connection pool timeout

## Bài nói

> Trong một scheduled job, Prisma bắt đầu báo timeout khi lấy database connection. Ban đầu em kiểm tra xem database có bị down hay query nào đặc biệt chậm không.
>
> Sau đó em xem flow của job và thấy job dùng `Promise.all` để xử lý nhiều shop cùng lúc. Mỗi shop lại tạo nhiều query nên tổng số query đồng thời tăng rất nhanh.
>
> Database connection pool có giới hạn. Khi tất cả connection đang bận, query mới phải chờ. Chờ quá lâu thì Prisma báo timeout.
>
> Em xử lý bằng cách giới hạn số shop chạy đồng thời thay vì start toàn bộ một lúc. Sau đó em theo dõi lại số lỗi và thời gian xử lý để kiểm tra hệ thống đã ổn định hơn chưa.

### Root cause là gì trong case này?

> Root cause là **job tạo quá nhiều query cùng lúc so với khả năng connection pool/database chịu được**. Prisma timeout chỉ là biểu hiện nhìn thấy bên ngoài.

### Tại sao không chỉ tăng pool?

> Nếu database còn capacity thì có thể tune pool. Nhưng nếu application vẫn tạo workload không giới hạn thì tăng pool chỉ đẩy điểm lỗi ra xa hơn. Em ưu tiên kiểm soát số task đồng thời trước rồi mới tune pool bằng metrics.

### Prevention là gì?

> Là những thay đổi giúp lỗi khó tái diễn: giới hạn concurrency, theo dõi query/connection usage, load test job và tránh `Promise.all` không giới hạn trên input lớn.

---

# Case 2 — Nginx 413 / upload timeout

## Bài nói

> Khi upload file lớn, request bị 413 hoặc timeout. Em không sửa ngay một config ngẫu nhiên mà đi theo đường request: client → Nginx → Node → storage.
>
> Với 413, em kiểm tra giới hạn request body ở Nginx/application. Với timeout, em xác định layer nào là bên đóng request vì chờ quá lâu.
>
> Nếu file cuối cùng được lưu ở S3 và backend không cần xử lý bytes, em ưu tiên flow client upload thẳng S3 bằng presigned URL thay vì để Node server chuyển tiếp toàn bộ file.

### “Layer” nghĩa là gì?

> Là từng thành phần request đi qua. Ví dụ browser là một layer, Nginx là một layer, Node application là một layer và S3 là một layer. Xác định đúng layer lỗi giúp mình không sửa sai chỗ.

### Bottleneck là gì?

> Là điểm giới hạn làm toàn flow chậm hoặc không scale được. Ví dụ nếu mọi file 500 MB đều phải đi qua Node server thì bandwidth của application server có thể trở thành bottleneck.

---

# Case 3 — Cron chạy nhiều lần

## Bài nói

> Khi application scale thành nhiều process hoặc instance, mỗi instance có thể cùng register một cron giống nhau. Một lịch chạy nhưng thực tế job lại chạy nhiều lần.
>
> Nếu job không chịu được việc chạy trùng thì có thể tạo duplicate data hoặc gọi external API nhiều lần.
>
> Em sẽ đảm bảo chỉ có một scheduler phù hợp hoặc dùng một cơ chế coordination chung. Đồng thời bản thân job cũng nên được thiết kế để nếu lỡ chạy lại thì không làm dữ liệu sai.

### Coordination là gì?

> Ở đây coordination chỉ là cách nhiều instance thống nhất “ai được chạy job này”. Ví dụ dùng một scheduler riêng, queue hoặc lock chung tùy hệ thống.

---

# Case 4 — External API timeout / 429

## Bài nói

> Khi external API lỗi, em phân loại lỗi trước thay vì retry mọi thứ.
>
> Timeout, 429 hoặc một số lỗi server có thể chỉ là tạm thời nên retry sau một khoảng có thể hợp lý. Nhưng input sai hoặc authentication sai thì retry y nguyên thường không giúp gì.
>
> Với 429, em đọc thông tin rate limit mà provider trả về và giảm tốc độ request. Với job lớn em dùng queue hoặc concurrency limit để tránh dồn request cùng lúc.

---

# Cách tìm nguyên nhân

### Em bắt đầu từ đâu?

> Em bắt đầu từ thời điểm lỗi xảy ra và bằng chứng có sẵn: log, metrics, error message, latency, trạng thái DB/API và deploy gần thời điểm đó.
>
> Sau đó em thu hẹp dần theo từng layer. Ví dụ lỗi request thì xem client/proxy/app/database thay vì cùng lúc thay đổi nhiều thứ.

### Evidence là gì?

> Là dữ liệu giúp chứng minh hoặc bác bỏ một giả thuyết, ví dụ log cho thấy connection timeout tăng đúng lúc cron start, hoặc metrics cho thấy active DB connections chạm ngưỡng tối đa.

### Hypothesis là gì?

> Là giả thuyết cần kiểm tra. Ví dụ “có thể database chậm” chỉ là hypothesis. Em cần log/metrics hoặc test để xác nhận chứ không coi đó là kết luận ngay.

### Metrics là gì?

> Là các số liệu theo dõi hệ thống theo thời gian, ví dụ request latency, error rate, số connection đang dùng, CPU, memory hoặc queue size.

### Correlation ID là gì?

> Là một ID đi theo cùng một request hoặc flow qua nhiều bước/service để mình tìm các log liên quan dễ hơn. Nếu hệ thống chưa có tracing đầy đủ thì correlation ID vẫn rất hữu ích.

### Tracing là gì?

> Là cách theo dõi một request đi qua nhiều service/operation và mất thời gian ở đâu. Khi phỏng vấn nếu chưa triển khai sâu, em chỉ cần nói đúng mục đích này thay vì cố kể tool mình chưa dùng.

---

# Fix xong chưa phải là hết

### Em kiểm tra fix thế nào?

> Em xem lại error rate, latency hoặc metric liên quan; chạy lại case đã gây lỗi nếu an toàn; và kiểm tra log để chắc flow mới hoạt động như mong đợi.

### Alert là gì?

> Là cảnh báo khi một metric hoặc điều kiện vượt ngưỡng để team biết sớm. Ví dụ tỷ lệ 5xx tăng mạnh hoặc queue backlog tăng liên tục.

### Runbook là gì?

> Là tài liệu ngắn hướng dẫn khi một loại sự cố xảy ra thì nên kiểm tra gì và xử lý bước đầu thế nào. Nếu chưa từng dùng runbook trong dự án thì không cần chủ động nhắc từ này khi phỏng vấn.

---

# Câu trả lời mẫu 60–90 giây

> Một vấn đề em từng gặp là Prisma timeout khi scheduled job chạy. Em kiểm tra log và flow của job, sau đó thấy `Promise.all` đang start xử lý nhiều shop cùng lúc, mỗi shop lại tạo nhiều database query.
>
> Khi số query tăng nhanh, toàn bộ connection trong pool bị sử dụng và query mới phải chờ đến timeout. Em chuyển sang giới hạn số shop được xử lý đồng thời rồi theo dõi lại error rate và thời gian job.
>
> Sau thay đổi, database ổn định hơn. Bài học của em là khi debug production cần tìm nguyên nhân gốc và kiểm tra capacity của cả hệ thống phía dưới, không chỉ nhìn error message ở application.

## Cách nhớ

`Thấy lỗi → xem ảnh hưởng → lấy log/metrics → đưa giả thuyết → kiểm tra → tìm nguyên nhân → sửa → đo lại → tránh tái diễn`
