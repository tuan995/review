# 10 — AWS S3 & Large File Upload

# Case: upload file lớn

## Bài nói

> Một case em từng gặp là upload file lớn qua backend. Khi file lên tới hàng trăm MB, request có thể bị `413 Request Entity Too Large`, timeout hoặc làm application server phải nhận và chuyển tiếp toàn bộ file nên tốn bandwidth và resource.
>
> Nếu flow là client → Node server → S3 thì backend trở thành điểm trung gian cho toàn bộ dữ liệu file. Với file lớn, em ưu tiên để backend chỉ kiểm tra user có quyền upload hay không rồi tạo một URL tạm thời cho phép client upload trực tiếp lên S3.
>
> Cách này giúp dữ liệu file không phải đi xuyên qua Node server. Backend vẫn kiểm soát quyền cấp upload, nhưng phần bytes đi thẳng từ client lên S3.
>
> Với file rất lớn hoặc mạng không ổn định, em cân nhắc multipart upload để chia file thành nhiều phần nhỏ hơn. Nếu một part lỗi thì retry part đó thay vì upload lại toàn bộ file.

---

# Interviewer đào sâu

### Presigned URL là gì?

> Là URL được backend tạo ra để cấp quyền tạm thời cho một operation cụ thể trên S3, ví dụ upload một object vào một key nhất định. Client dùng URL đó trong thời gian giới hạn mà không cần biết AWS secret key.

### Vì sao presigned URL an toàn hơn việc gửi AWS key cho client?

> Vì client chỉ nhận quyền tạm thời cho operation đã được ký sẵn. AWS secret key vẫn nằm ở server. Em vẫn phải giới hạn thời gian sống của URL, object key và chỉ cấp URL sau khi đã kiểm tra quyền của user.

### `413 Request Entity Too Large` nghĩa là gì?

> Nghĩa là một layer nhận request, thường là proxy hoặc application server, từ chối vì request body lớn hơn giới hạn cấu hình. Với Nginx em sẽ kiểm tra `client_max_body_size`.

### Timeout là gì trong case upload?

> Là một layer chờ quá lâu mà request chưa hoàn thành nên chủ động đóng request. Em cần xác định timeout nằm ở client, Nginx, Node hay service/storage phía sau chứ không tăng mọi timeout một cách mù quáng.

### Multipart upload là gì?

> Là chia một file lớn thành nhiều part rồi upload từng part lên S3. Khi tất cả part thành công thì hoàn tất upload. Lợi ích là retry riêng part bị lỗi và có thể upload nhiều part song song với mức concurrency phù hợp.

### Làm sao biết upload đã xong?

> Có vài cách tùy flow. Client có thể gọi API `complete/confirm` sau khi upload. Backend có thể kiểm tra object đã tồn tại và metadata phù hợp bằng S3 API. Hoặc hệ thống có thể dùng S3 event để báo khi object được tạo.

### Metadata là gì?

> Là thông tin mô tả object ngoài nội dung file, ví dụ kích thước, content type, key, thời gian upload hoặc custom metadata mình lưu kèm.

### Nếu upload S3 xong nhưng DB chưa lưu metadata thì sao?

> Đây là tình huống hai hệ thống cập nhật không cùng lúc. S3 đã có file nhưng database chưa có record tương ứng.
>
> Em có thể upload file vào trạng thái/key tạm, sau đó chỉ đánh dấu hoàn tất khi backend xác nhận cả S3 và DB đã đúng. Ngoài ra có thể có cleanup job để xóa những file đã upload nhưng không bao giờ được hoàn tất.

### “Distributed consistency problem” là gì?

> Nếu dùng từ này, em muốn nói **một nghiệp vụ đi qua nhiều hệ thống nhưng không có một transaction chung để rollback tất cả cùng lúc**. Trong ví dụ này, database rollback không thể tự xóa file đã upload lên S3.

### Orphan object là gì?

> Là file/object còn nằm trong S3 nhưng không còn record hợp lệ trong hệ thống để tham chiếu tới nó. Cleanup job có thể tìm và xóa những object kiểu này theo rule an toàn.

---

# Tại sao không chỉ tăng Nginx limit?

> Nếu hệ thống vẫn cần upload qua backend thì em phải cấu hình đúng `client_max_body_size` và timeout. Nhưng nếu business cho phép direct upload thì chỉ tăng limit chưa giải quyết việc backend vẫn phải chuyển tiếp hàng trăm MB cho mỗi request.
>
> Em ưu tiên sửa flow để application server không nằm trong đường truyền file lớn nếu nó không cần xử lý bytes đó.

---

# STAR version

**Situation**

> File lớn upload qua application/Nginx bị giới hạn kích thước hoặc timeout.

**Task**

> Cho phép upload ổn định mà không làm Node server phải gánh toàn bộ file.

**Action**

> Em kiểm tra giới hạn Nginx/server, sau đó với flow phù hợp em chuyển sang backend cấp presigned URL để client upload thẳng S3. Với file lớn em cân nhắc multipart upload để retry từng phần.

**Result**

> Giảm bandwidth/resource đi qua application server và việc retry file lớn linh hoạt hơn.

## Cách nhớ

`Client xin quyền → backend kiểm tra → cấp URL tạm → client upload thẳng S3 → backend xác nhận → cleanup file không hoàn tất`
