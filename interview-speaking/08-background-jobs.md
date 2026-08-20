# 08 — Background Job, Cron & Queue

# 1. Khi nào dùng background job?

## Bài nói

> Em đưa công việc ra background khi task không cần hoàn thành ngay trong request hiện tại, chạy khá lâu hoặc cần retry độc lập.
>
> Ví dụ user bấm đồng bộ dữ liệu cho nhiều shop. API không nhất thiết phải giữ request mở vài phút để chờ toàn bộ job xong. Backend có thể tạo job rồi trả response sớm, sau đó worker xử lý phía sau.
>
> Cách này giúp API phản hồi nhanh hơn và job có thể retry riêng nếu lỗi.

### Background job là gì?

> Là công việc chạy phía sau request chính. User không cần giữ kết nối tới API cho đến khi toàn bộ task hoàn thành.

### Worker là gì?

> Worker là process hoặc service lấy job ra và thực hiện công việc đó. Ví dụ queue có 100 job đồng bộ shop thì nhiều worker có thể lần lượt lấy job để xử lý.

---

# 2. Cron và Queue khác nhau thế nào?

> Cron chủ yếu trả lời câu **“khi nào bắt đầu công việc?”**, ví dụ mỗi ngày 2 giờ sáng chạy job kiểm tra dữ liệu.
>
> Queue chủ yếu giúp **“các task được xếp hàng và worker xử lý thế nào?”**, ví dụ retry, giới hạn số job chạy cùng lúc và giữ lại job khi worker restart.
>
> Hai thứ có thể kết hợp: cron chỉ có nhiệm vụ tạo các job vào queue, còn worker xử lý từng job.

### Tại sao không để cron xử lý toàn bộ luôn?

> Nếu cron một lần xử lý hàng nghìn item trong memory rồi process chết giữa chừng thì khó biết item nào đã xong. Với queue, từng job có trạng thái rõ hơn và có thể retry riêng.

---

# 3. Cron chạy trùng khi có nhiều instance

## Bài nói

> Khi application chạy nhiều process hoặc container, mỗi instance có thể cùng đăng ký một cron giống nhau. Kết quả là đúng một thời điểm nhưng job lại chạy nhiều lần.
>
> Nếu job gửi email hoặc gọi API bên ngoài thì duplicate execution có thể gây tác dụng phụ.
>
> Với job chỉ nên có một scheduler, em có thể tách scheduler riêng hoặc dùng cơ chế lock chung để chỉ một instance được quyền chạy tại một thời điểm. Nếu dùng queue thì scheduler có thể tạo một job unique để tránh tạo trùng.

### Distributed lock là gì?

> Là một lock được lưu ở nơi nhiều instance cùng nhìn thấy, ví dụ Redis hoặc database. Mục tiêu là chỉ một instance giữ quyền thực hiện một công việc tại một thời điểm.

### Vì sao lock này dễ bị hỏi sâu?

> Vì nếu process giữ lock rồi chết, mình phải đảm bảo lock có thời gian hết hạn hoặc cơ chế ownership phù hợp. Nếu không, job có thể bị khóa mãi. Khi interview nếu chưa implement sâu thì em nói rõ mình hiểu mục tiêu và các rủi ro này, không cần khẳng định một thuật toán lock phức tạp.

---

# 4. Retry và chạy trùng

> Background job phải giả định có thể được chạy lại. Ví dụ worker xử lý xong nhưng chết trước khi báo queue rằng job đã hoàn thành, queue có thể giao job đó lại cho worker khác.
>
> Vì vậy operation quan trọng nên được thiết kế để chạy lại không gây duplicate data hoặc duplicate external action.

### Acknowledge là gì?

> Là tín hiệu worker gửi cho queue để báo rằng job đã xử lý xong. Nếu queue không nhận được tín hiệu đó, tùy cơ chế nó có thể cho job chạy lại.

### Idempotent là gì?

> Nghĩa là chạy lại cùng một job nhưng kết quả cuối không bị nhân đôi hoặc sai thêm. Ví dụ cùng job tạo mapping cho một order chạy hai lần nhưng database vẫn chỉ có một mapping đúng.

### Dead-letter queue là gì?

> Là nơi giữ những job đã fail quá nhiều lần thay vì cứ retry vô hạn. Mục đích là để mình kiểm tra nguyên nhân hoặc chạy lại sau khi đã sửa lỗi.

> Khi nói phỏng vấn, nếu thấy từ `dead-letter queue` khó nhớ thì có thể nói đơn giản: **“sau một số lần retry, em chuyển job lỗi sang một nhóm riêng để kiểm tra thay vì retry mãi.”**

---

# 5. Theo dõi queue như thế nào?

> Em không chỉ log `job started`. Em muốn biết còn bao nhiêu job đang chờ, job cũ nhất đã chờ bao lâu, một job mất bao lâu để xử lý, tỷ lệ thành công/thất bại và số lần retry.

### Queue depth là gì?

> Là số lượng job đang chờ trong queue. Nếu số này tăng liên tục thì tốc độ tạo job đang lớn hơn tốc độ worker xử lý.

### Processing latency là gì?

> Là thời gian job mất để được xử lý. Có thể tách thời gian chờ trong queue và thời gian worker thực sự chạy job nếu cần phân tích sâu hơn.

### Backlog là gì?

> Là lượng công việc tồn đọng chưa xử lý xong. Nếu backlog ngày càng tăng thì worker không theo kịp workload.

---

# Bài nói 60–90 giây

> Em dùng background job cho những task dài hoặc không cần trả kết quả ngay trong request, ví dụ đồng bộ dữ liệu nhiều shop.
>
> Nếu job chạy theo lịch thì cron có thể là điểm bắt đầu. Nhưng với workload lớn em thích để cron chỉ tạo job vào queue, sau đó worker xử lý với số lượng đồng thời có giới hạn.
>
> Queue giúp em retry từng job và giữ trạng thái tốt hơn nếu worker restart. Em cũng thiết kế job để có thể chạy lại mà không tạo duplicate data, vì trong thực tế queue có thể giao lại cùng một job.
>
> Khi chạy nhiều application instance, em còn phải tránh trường hợp mỗi instance cùng chạy một cron. Em sẽ dùng một scheduler rõ ràng hoặc cơ chế coordination phù hợp.

## Cách nhớ

`Cron = khi nào chạy → Queue = xếp việc → Worker = làm việc → Retry → tránh duplicate → theo dõi job tồn`
