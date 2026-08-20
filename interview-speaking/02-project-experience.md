# 02 — Project Experience: Shopify & Backend Integration

## Cách giới thiệu dự án

> Trong một số dự án gần đây em phát triển backend cho các ứng dụng liên quan tới Shopify. Backend của em nhận dữ liệu từ Shopify, lưu những phần hệ thống cần dùng vào database, xử lý webhook và các job chạy phía sau, sau đó cung cấp API cho frontend.
>
> Điểm khó của dạng dự án này là dữ liệu nằm ở nhiều hệ thống khác nhau. Shopify có thể thay đổi trước, database của mình cập nhật sau; API bên ngoài có thể chậm, timeout hoặc giới hạn số request. Vì vậy ngoài CRUD, em phải nghĩ tới cách đồng bộ dữ liệu, retry khi lỗi và tránh xử lý cùng một sự kiện nhiều lần gây duplicate data.

---

# Case 1 — Tại sao lưu database thay vì gọi Shopify liên tục?

## Bài nói

> Có những dữ liệu em có thể lấy trực tiếp từ Shopify API. Nhưng nếu mỗi lần frontend cần hiển thị dữ liệu lại gọi Shopify thì response của hệ thống phụ thuộc vào tốc độ và tình trạng của Shopify. Khi traffic tăng, số lần gọi cũng tăng và dễ chạm giới hạn API.
>
> Vì vậy với dữ liệu cần đọc thường xuyên, em lưu những field cần thiết vào database của hệ thống. Request thông thường đọc từ database của mình, còn Shopify vẫn là nơi giữ dữ liệu chính thức đối với dữ liệu do Shopify quản lý.
>
> Khi Shopify thay đổi, hệ thống local cập nhật thông qua webhook hoặc job đồng bộ. Đổi lại, dữ liệu trong database của mình có thể chậm hơn Shopify một khoảng ngắn. Với dữ liệu quan trọng như inventory, em cần cơ chế cập nhật nhanh và có job kiểm tra lại định kỳ.

### “Source of truth” là gì?

> Nếu em dùng từ này, em muốn nói **nơi được coi là dữ liệu chính thức khi hai hệ thống khác nhau**. Ví dụ với inventory do Shopify quản lý, nếu Shopify và database local lệch nhau thì em coi Shopify là nguồn chính để sửa lại database local.

### “Stale data” là gì?

> Là dữ liệu local chưa kịp cập nhật theo dữ liệu mới nhất. Ví dụ Shopify đã đổi stock từ 10 xuống 5 nhưng database của mình vẫn đang là 10 trong vài phút.

### Tại sao không gọi Shopify trực tiếp?

> Vì response sẽ phụ thuộc external API, latency thường cao hơn đọc local database, khó query/report theo nhu cầu riêng và dễ chạm rate limit khi traffic tăng.

### Nhược điểm của lưu local database?

> Mình phải tự giải quyết chuyện dữ liệu cập nhật chậm, webhook gửi trùng, webhook bị miss và cách kiểm tra lại dữ liệu định kỳ.

---

# Case 2 — Inventory thay đổi liên tục nhưng job chỉ chạy mỗi ngày

## Bài nói

> Nếu inventory thay đổi liên tục mà hệ thống chỉ đồng bộ một lần mỗi ngày thì dữ liệu có thể sai gần 24 giờ. Với tồn kho thì khoảng trễ đó thường quá lớn.
>
> Cách đơn giản là tăng cron lên vài phút một lần, nhưng như vậy hệ thống phải liên tục gọi Shopify kể cả khi không có gì thay đổi, vừa tốn API request vừa vẫn có độ trễ.
>
> Vì vậy em ưu tiên webhook cho những thay đổi cần cập nhật nhanh. Khi inventory thay đổi, Shopify chủ động báo cho hệ thống của mình và mình cập nhật database local.
>
> Em vẫn giữ một job chạy định kỳ để kiểm tra lại. Lý do là webhook có thể bị lỗi, bị miss hoặc xử lý thất bại. Job này sẽ so sánh lại dữ liệu và sửa những record bị lệch.

### “Reconciliation job” là gì?

> Là job **kiểm tra và sửa lại dữ liệu bị lệch giữa hai hệ thống**. Ví dụ mỗi đêm lấy lại một phần inventory từ Shopify, so với local DB và cập nhật những record không khớp.

### Tại sao vẫn cần cron nếu đã có webhook?

> Webhook giúp cập nhật nhanh. Cron kiểm tra lại giúp sửa các trường hợp webhook không tới hoặc xử lý lỗi. Hai cơ chế bổ sung cho nhau.

### Nếu webhook gửi trùng thì sao?

> Em thiết kế handler để cùng một event xử lý lại không tạo kết quả sai. Ví dụ lưu event ID đã xử lý hoặc dùng unique constraint/business key để không tạo record lần hai.

### “Idempotent” là gì?

> Idempotent nghĩa là cùng một operation bị chạy lại nhưng kết quả cuối không bị nhân đôi hoặc sai thêm. Ví dụ cùng webhook `order.created` tới hai lần nhưng hệ thống vẫn chỉ có một order tương ứng.

### Nếu event đến sai thứ tự?

> Nếu thứ tự ảnh hưởng kết quả, em không áp dụng mù event cũ. Em có thể so sánh thời gian/version hoặc lấy lại trạng thái mới nhất từ Shopify rồi cập nhật theo trạng thái hiện tại.

---

# Case 3 — External API failure

## Bài nói

> Khi tích hợp API bên ngoài, em coi lỗi mạng hoặc lỗi tạm thời là chuyện có thể xảy ra bình thường. Request có thể timeout, bị giới hạn request hoặc phía provider trả lỗi server.
>
> Với lỗi có khả năng hết sau một thời gian, em retry nhưng có giới hạn. Em không retry ngay liên tục mà chờ một khoảng rồi mới thử lại. Nếu lần sau vẫn lỗi thì khoảng chờ có thể tăng dần.
>
> Với lỗi do input sai hoặc authentication sai thì retry nhiều lần không giúp gì. Trường hợp đó em log rõ nguyên nhân, cảnh báo nếu cần và đưa job/request về trạng thái failed để xử lý đúng lỗi.

### “Transient error” là gì?

> Là lỗi có khả năng chỉ xảy ra tạm thời, ví dụ timeout mạng hoặc server bên ngoài đang quá tải. Thử lại sau có thể thành công.

### “Backoff” là gì?

> Là khi retry thì không thử lại ngay lập tức. Mình chờ một khoảng rồi mới thử lại, thường các lần sau chờ lâu hơn để tránh tiếp tục gây áp lực lên hệ thống đang lỗi.

### Retry bao nhiêu lần?

> Em không dùng một con số cố định cho mọi API. Em dựa vào mức độ quan trọng của operation, policy của provider và thời gian user/job có thể chờ. Quan trọng là retry có giới hạn chứ không lặp vô hạn.

### Retry POST có nguy hiểm không?

> Có. Request đầu tiên có thể đã thành công ở server nhưng response bị mất trên đường về. Nếu mình gửi lại `create payment` hoặc `create order` một cách mù quáng thì có thể tạo hai record.
>
> Với operation kiểu này em dùng idempotency key hoặc business key để server nhận ra đây là cùng một request được gửi lại.

## Cách nhớ chương

`Không gọi Shopify mọi lúc → lưu local → webhook cập nhật nhanh → cron kiểm tra lại → tránh duplicate → retry có giới hạn`
