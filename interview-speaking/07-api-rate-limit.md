# 07 — API Integration & Rate Limit

# Case: Shopify / Google / Third-party API

## Bài nói

> Khi tích hợp API bên ngoài, em luôn để ý giới hạn số request mà provider cho phép trong một khoảng thời gian. Nếu mỗi request của user lại gọi thẳng external API hoặc một background job cùng lúc xử lý quá nhiều item thì hệ thống rất dễ chạm giới hạn đó.
>
> Em thường xử lý theo nhiều lớp. Đầu tiên là giảm những request không cần thiết bằng database local hoặc cache. Sau đó em giới hạn số task đang chạy cùng lúc hoặc số request gửi đi trong một khoảng thời gian.
>
> Nếu provider trả `429 Too Many Requests`, em không retry ngay liên tục. Nếu API trả `Retry-After` hoặc thông tin tương tự thì em ưu tiên làm theo. Nếu không có, em chờ một khoảng rồi mới thử lại và khoảng chờ có thể tăng dần ở các lần retry tiếp theo.
>
> Với job không cần kết quả ngay, em có thể dùng queue để trải workload ra theo thời gian thay vì dồn toàn bộ request vào một lúc.

### Rate limit là gì?

> Rate limit là giới hạn số request được phép gửi trong một khoảng thời gian, ví dụ một số lượng request mỗi giây/phút hoặc theo cơ chế quota riêng của provider.

### Concurrency limit khác rate limit thế nào?

> Concurrency limit giới hạn **bao nhiêu operation đang chạy cùng lúc**. Rate limit giới hạn **bao nhiêu operation được gửi trong một khoảng thời gian**.
>
> Ví dụ em có thể chỉ cho 5 request chạy cùng lúc nhưng nếu mỗi request hoàn thành rất nhanh thì tổng số request/phút vẫn có thể lớn. Vì vậy có hệ thống cần cả hai loại giới hạn.

### 429 xử lý sao?

> Em đọc thông tin retry mà provider trả về nếu có. Sau đó retry có giới hạn và chờ giữa các lần retry. Em cũng xem có cần giảm tốc độ worker hoặc giảm lượng request phát sinh từ đầu hay không.

### Backoff là gì?

> Là thay vì lỗi xong thử lại ngay, em chờ một khoảng rồi mới retry. Các lần sau có thể chờ lâu hơn để tránh tiếp tục gây tải lên service đang bận.

### Jitter là gì?

> Nếu nhiều worker đều lỗi cùng lúc và đều chờ đúng 2 giây rồi retry thì 2 giây sau chúng lại cùng dồn request lên provider. Jitter là thêm một khoảng ngẫu nhiên nhỏ vào thời gian chờ để các worker retry lệch nhau.

### “Thundering herd” là gì?

> Đây là tên gọi cho tình huống rất nhiều worker/request cùng thức dậy hoặc retry gần như cùng lúc, tạo một đợt tải lớn. Khi nói phỏng vấn em có thể tránh từ này và nói thẳng “nhiều worker retry cùng thời điểm”.

### Retry mọi lỗi không?

> Không. Timeout, 429 hoặc một số 5xx có thể là lỗi tạm thời nên retry có thể hợp lý. Nhưng lỗi input sai hoặc authentication sai thì retry y nguyên nhiều lần thường không giúp gì.

### “Transient error” là gì?

> Là lỗi có khả năng chỉ xảy ra tạm thời, ví dụ network timeout hoặc server bên ngoài đang quá tải. Thử lại sau có thể thành công.

### “Bounded retry” là gì?

> Là retry có giới hạn số lần hoặc giới hạn thời gian. Em tránh retry vô hạn vì nó có thể biến một lỗi bên ngoài thành workload lớn hơn cho chính hệ thống của mình.

### Cache có giải quyết rate limit hoàn toàn không?

> Không. Cache chủ yếu giúp giảm những request đọc lặp lại. Dữ liệu cache cũng có thể cũ và cần cơ chế cập nhật/xóa. Những operation ghi hoặc đồng bộ vẫn cần kiểm soát tốc độ.

---

# Bài nói 60 giây

> Khi làm với Shopify hoặc API bên thứ ba, em không giả định mình có thể gọi API không giới hạn. Em giảm request không cần thiết bằng local DB/cache, giới hạn số task chạy cùng lúc và với job lớn thì đưa qua queue để điều tiết tốc độ.
>
> Khi gặp 429, em ưu tiên đọc `Retry-After` hoặc metadata của provider. Nếu cần retry thì em chờ giữa các lần và có giới hạn số lần thử. Em cũng tránh để nhiều worker retry đúng cùng thời điểm bằng cách thêm một khoảng ngẫu nhiên nhỏ vào thời gian chờ.
>
> Quan trọng là không chỉ xử lý 429 sau khi nó xảy ra mà phải thiết kế để request rate ổn định ngay từ đầu.

## Cách nhớ

`Giảm request → giới hạn số task/tốc độ → 429 thì chờ → retry có giới hạn → không retry tất cả lỗi`
