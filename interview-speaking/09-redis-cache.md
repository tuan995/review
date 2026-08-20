# 09 — Redis & Caching

# 1. Khi nào em dùng Redis?

## Bài nói

> Em dùng Redis khi cần lưu dữ liệu truy cập nhanh hoặc dữ liệu chỉ cần tồn tại trong một khoảng thời gian, ví dụ cache, trạng thái tạm thời, giới hạn request hoặc một số cơ chế coordination giữa nhiều instance.
>
> Nhưng em không thêm Redis chỉ vì Redis nhanh. Đầu tiên em kiểm tra bottleneck thật sự nằm ở đâu. Nếu database đã đủ nhanh hoặc dữ liệu thay đổi liên tục và rất khó cache đúng thì thêm cache có thể làm hệ thống phức tạp hơn mà lợi ích không nhiều.

### Cache là gì?

> Cache là nơi giữ tạm một bản dữ liệu để lần sau đọc nhanh hơn hoặc tránh phải gọi lại database/external API. Dữ liệu gốc vẫn thường nằm ở database hoặc hệ thống chính.

### TTL là gì?

> TTL là thời gian một key được giữ trước khi tự hết hạn. Ví dụ cache product trong 5 phút thì sau 5 phút Redis có thể xóa key đó và request sau phải lấy lại dữ liệu mới.

---

# 2. Cache-aside

## Bài nói

> Cách em thường dùng là application đọc Redis trước. Nếu Redis có dữ liệu thì trả luôn. Nếu không có thì đọc database, sau đó lưu kết quả vào Redis để những lần đọc tiếp theo nhanh hơn.
>
> Khi dữ liệu trong database thay đổi, em cần xóa hoặc cập nhật cache để tránh user tiếp tục đọc dữ liệu cũ.

```text
Request
   |
   v
Redis có dữ liệu? ---- có ----> trả kết quả
   |
  không
   v
Database → lưu lại Redis → trả kết quả
```

### Cache-aside là gì?

> Là pattern mà application tự chịu trách nhiệm đọc cache trước, cache miss thì đọc source chính rồi tự ghi lại cache. Khi phỏng vấn có thể nói flow ở trên trước, sau đó mới gọi tên là cache-aside.

### Cache miss/hit là gì?

> Cache hit là tìm thấy dữ liệu trong cache. Cache miss là không tìm thấy nên phải lấy từ database hoặc nguồn khác.

---

# 3. Cache invalidation

> Phần khó nhất của cache thường không phải đọc mà là **khi nào xóa hoặc cập nhật dữ liệu cũ**.
>
> Ví dụ product price vừa đổi trong database nhưng Redis vẫn giữ giá cũ. Em có thể xóa cache ngay sau khi update DB, cập nhật lại cache hoặc dùng TTL như một lớp an toàn để dữ liệu cũ không tồn tại mãi.

### Invalidation là gì?

> Là làm cho cache cũ không còn được dùng nữa, thường bằng cách xóa key hoặc thay bằng dữ liệu mới.

### Chọn TTL bao nhiêu?

> Không có một con số đúng cho mọi dữ liệu. Em nhìn mức độ user có thể chấp nhận dữ liệu cũ trong bao lâu, dữ liệu được đọc nhiều đến đâu và chi phí để lấy lại dữ liệu từ source chính.

---

# 4. Nhiều request cùng cache miss

> Một tình huống dễ gặp là một key được đọc rất nhiều vừa hết hạn. Hàng trăm request cùng lúc không thấy cache và cùng query database. Database có thể bị một đợt tải lớn dù bình thường cache giúp giảm tải.

### Cache stampede là gì?

> Đó chính là tình huống nhiều request cùng cache miss và cùng đổ xuống database/source phía dưới. Khi nói phỏng vấn em có thể nói thẳng hiện tượng trước, không nhất thiết phải mở đầu bằng thuật ngữ này.

### Có thể xử lý thế nào?

> Tùy mức độ, em có thể để chỉ một request đi lấy dữ liệu mới còn request khác chờ, làm TTL của các key lệch nhau một chút, hoặc tạm dùng dữ liệu cũ trong lúc một request cập nhật cache mới.

### “Single-flight” nghĩa là gì?

> Nghĩa là khi nhiều request cùng cần build lại một cache key, chỉ một request thực sự đi lấy dữ liệu, các request khác chờ dùng cùng kết quả đó.

### “Stale-while-revalidate” nghĩa là gì?

> Có thể hiểu là tạm trả dữ liệu cache cũ trong một khoảng ngắn, đồng thời có một process/request đi cập nhật dữ liệu mới ở phía sau. Cách này chỉ phù hợp nếu business chấp nhận dữ liệu cũ trong khoảng thời gian đó.

---

# 5. Redis down thì sao?

> Nếu Redis chỉ là cache, em cố thiết kế để application vẫn có thể đọc từ database khi Redis lỗi. Nhưng em cũng phải cẩn thận: nếu toàn bộ traffic đột ngột chuyển xuống database thì database có thể bị quá tải.
>
> Nếu Redis đang giữ dữ liệu quan trọng chứ không chỉ cache, ví dụ queue hoặc state cần bền vững, thì yêu cầu về persistence, replication và failover sẽ khác và em cần thiết kế riêng cho vai trò đó.

### Fallback là gì?

> Là đường xử lý thay thế khi component chính không dùng được. Ví dụ Redis lỗi thì application tạm đọc trực tiếp từ database.

### Source of truth là gì trong cache?

> Là nơi mình coi là dữ liệu chính thức. Với cache-aside thông thường, database là dữ liệu chính; Redis chỉ giữ bản copy để đọc nhanh hơn.

---

# 6. Bài nói 60 giây

> Khi dùng Redis làm cache, em bắt đầu từ việc xác định dữ liệu nào đọc nhiều và có thể chấp nhận chậm cập nhật một khoảng ngắn.
>
> Application đọc Redis trước; nếu không có thì đọc database rồi ghi lại cache với TTL. Khi database thay đổi, em xóa hoặc cập nhật cache để giảm khả năng trả dữ liệu cũ.
>
> Em cũng để ý trường hợp nhiều request cùng cache miss vì một key vừa expire, vì lúc đó toàn bộ request có thể cùng đổ xuống database. Nếu cần em sẽ giới hạn việc rebuild cache hoặc cho một request cập nhật còn request khác dùng kết quả chung.

## Cách nhớ

`Có thực sự cần cache? → đọc Redis trước → miss thì đọc DB → update/xóa cache khi data đổi → tránh nhiều request cùng miss → có fallback khi Redis lỗi`
