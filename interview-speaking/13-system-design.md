# 13 — System Design Speaking Framework

Mục tiêu của chương này không phải học thuộc một architecture. Mục tiêu là **biết nói suy nghĩ của mình thành từng bước và giải thích tại sao thêm từng component**.

# 1. Bắt đầu bằng yêu cầu

## Bài nói

> Trước khi thiết kế, em muốn hiểu hệ thống cần làm gì, có bao nhiêu user/request dự kiến, dữ liệu lớn tới mức nào và phần nào quan trọng nhất.
>
> Ví dụ có hệ thống ưu tiên response nhanh, có hệ thống ưu tiên dữ liệu phải đúng tuyệt đối, có hệ thống lại cần tiếp tục hoạt động ngay cả khi một service lỗi. Nếu chưa biết yêu cầu thì em chưa muốn thêm cache, queue hay chia database ngay.

### Functional requirement là gì?

> Là hệ thống **phải làm được chức năng gì**, ví dụ user tạo order, xem order, upload file hoặc đồng bộ product.

### Non-functional requirement là gì?

> Là yêu cầu về cách hệ thống vận hành, ví dụ response cần nhanh tới mức nào, chịu được bao nhiêu request, downtime chấp nhận được không và dữ liệu có thể chậm cập nhật bao lâu.

### Latency là gì?

> Là thời gian từ lúc request bắt đầu tới lúc có kết quả. Nói đơn giản là user phải chờ bao lâu.

### Availability là gì?

> Là mức độ hệ thống có thể tiếp tục phục vụ request khi có lỗi. Khi nói phỏng vấn em có thể nói đơn giản “hệ thống có cần luôn sẵn sàng hay có thể chấp nhận downtime ngắn”.

### Correctness là gì?

> Là kết quả nghiệp vụ có đúng không. Ví dụ payment không được ghi nhận hai lần, stock không được bán âm hoặc order phải có dữ liệu đầy đủ.

---

# 2. Vẽ flow đơn giản trước

> Em thường bắt đầu từ flow tối thiểu:

```text
Client
  |
API
  |
Business logic
  |
Database
```

> Sau đó em mới hỏi từng điểm có vấn đề gì. Nếu traffic tăng mới nghĩ tới nhiều API instance/load balancer. Nếu đọc DB quá nhiều mới cân nhắc cache. Nếu task dài mới cân nhắc queue.

### Load balancer là gì?

> Là thành phần nhận request rồi phân phối request sang nhiều application instance. Mục tiêu là không để tất cả traffic chỉ đi vào một server.

### Instance là gì?

> Có thể hiểu là một bản application đang chạy. Ví dụ mình chạy 3 Node.js process/container cùng phục vụ một API thì có 3 instance.

---

# 3. Chọn database

## Bài nói

> Em chọn database dựa trên loại dữ liệu và cách hệ thống cần truy cập, không chọn chỉ vì quen dùng.
>
> Nếu dữ liệu có quan hệ rõ, cần transaction giữa nhiều record và query relational nhiều thì PostgreSQL là lựa chọn em quen dùng. Với dữ liệu dạng document linh hoạt và access pattern phù hợp, MongoDB có thể hợp lý hơn.

### Access pattern là gì?

> Là cách application thường đọc/ghi dữ liệu. Ví dụ query thường lấy order theo `shop + createdAt`, hoặc tìm product theo `shop + productId`. Thiết kế schema/index nên phục vụ những cách truy cập thực tế đó.

### Relational database là gì?

> Là database tổ chức dữ liệu thành bảng có quan hệ với nhau và thường hỗ trợ SQL/transaction mạnh, ví dụ PostgreSQL/MySQL.

### Document database là gì?

> Là database lưu dữ liệu theo document, thường gần với object/JSON hơn, ví dụ MongoDB. Không có nghĩa MongoDB luôn tốt hơn cho dữ liệu linh hoạt; vẫn phải xem query và consistency requirement.

---

# 4. Khi nào thêm cache?

> Em thêm cache khi có dữ liệu đọc nhiều, tốn thời gian/tài nguyên để lấy lại và business chấp nhận dữ liệu có thể cũ trong một khoảng ngắn.
>
> Nếu thêm Redis, em phải trả lời tiếp: cache key là gì, giữ bao lâu và khi dữ liệu gốc thay đổi thì cache được xóa/cập nhật thế nào.

### Hot path là gì?

> Nếu dùng từ này, em muốn nói **luồng được gọi rất thường xuyên hoặc ảnh hưởng trực tiếp tới latency của user**. Khi phỏng vấn có thể nói thẳng “API này bị gọi rất nhiều” thay vì dùng `hot path`.

---

# 5. Khi nào thêm queue?

> Em thêm queue khi task không cần xử lý ngay trong request, cần retry độc lập hoặc workload có thể dồn lên theo đợt.
>
> Ví dụ cron tạo 100 nghìn task đồng bộ. Thay vì một process xử lý tất cả cùng lúc, em đưa từng task vào queue rồi để worker lấy ra với concurrency giới hạn.

### “Decouple” là gì?

> Nếu dùng từ này, em muốn nói **tách việc tạo task khỏi việc xử lý task**. API chỉ cần tạo job, worker xử lý sau. Khi nói phỏng vấn em ưu tiên giải thích flow này trước rồi mới gọi là decouple.

### “Absorb traffic burst” là gì?

> Nghĩa là khi task đến dồn dập, queue giữ chúng lại để worker xử lý dần thay vì ép downstream phải xử lý tất cả ngay lập tức.

### Eventual consistency là gì?

> Nghĩa là dữ liệu giữa các phần của hệ thống có thể chưa giống nhau ngay lập tức nhưng sau một khoảng thời gian sẽ được đồng bộ về trạng thái đúng.
>
> Ví dụ webhook tới → event vào queue → worker vài giây sau mới update database. Trong vài giây đó dữ liệu local chưa phải mới nhất.

---

# 6. External API

## Bài nói

> Với Shopify, Stripe hoặc Google, em coi service bên ngoài là dependency mình không kiểm soát hoàn toàn. Nó có thể timeout, giới hạn request hoặc trả lỗi.
>
> Vì vậy em nghĩ trước về timeout, retry có giới hạn, tránh duplicate khi retry và cách kiểm tra lại trạng thái nếu webhook/event bị miss.

### Dependency là gì?

> Là một thành phần mà hệ thống mình phụ thuộc để hoàn thành công việc. Ví dụ nếu API của mình cần gọi Shopify thì Shopify là external dependency của flow đó.

### Circuit breaker là gì?

> Nếu interviewer hỏi sâu: đây là pattern tạm ngừng gọi một service đang lỗi quá nhiều trong một khoảng thời gian để tránh tiếp tục dồn request vào nó.
>
> Nếu chưa từng implement thì em không chủ động đưa từ này vào câu trả lời. Chỉ cần nói mình giới hạn retry và tránh tiếp tục spam service đang lỗi.

---

# 7. Khi traffic tăng thì scale thế nào?

## Bài nói

> Em không nhảy ngay tới sharding. Đầu tiên em đo xem bottleneck nằm ở đâu: CPU của API, query DB, connection pool, external API hay queue worker.
>
> Nếu API không giữ state riêng trong memory và có thể chạy nhiều bản giống nhau, em có thể tăng số instance sau load balancer.
>
> Nếu database là bottleneck, em kiểm tra query/index trước. Sau đó mới cân nhắc các bước lớn hơn như read replica hoặc partition dữ liệu tùy trường hợp.

### Stateless API là gì?

> Là API không phụ thuộc vào state chỉ nằm trong memory của riêng một instance giữa các request. Ví dụ session quan trọng nằm ở shared storage/database thay vì chỉ ở RAM của server A.
>
> Nhờ vậy request tiếp theo có thể đi vào server B mà vẫn xử lý được.

### Horizontal scale là gì?

> Là tăng số instance/server để chia tải. Ví dụ từ 2 API instances lên 6 instances.

### Vertical scale là gì?

> Là tăng tài nguyên của một server hiện tại, ví dụ thêm CPU/RAM.

### Read replica là gì?

> Là bản sao database chủ yếu dùng để phục vụ read. Nó có thể giảm tải read cho primary, nhưng dữ liệu replica có thể chậm hơn primary một chút tùy cơ chế replication.

### Partitioning là gì?

> Là chia dữ liệu/workload thành nhiều phần theo một tiêu chí, ví dụ theo shop hoặc khoảng thời gian, để không phải mọi thứ cùng dồn vào một chỗ.

### Sharding là gì?

> Là một dạng chia dữ liệu ra nhiều database/node độc lập theo shard key. Nó tăng độ phức tạp đáng kể nên em không chọn sớm nếu chưa có bottleneck thật sự yêu cầu.

---

# 8. Reliability — luôn hỏi “nếu lỗi thì sao?”

Khi thiết kế, em tự hỏi:

- Worker chết giữa job thì job có mất không?
- Webhook gửi hai lần thì có duplicate data không?
- Database update thành công nhưng external call fail thì trạng thái xử lý thế nào?
- Redis down thì API còn chạy được không?
- External API trả 429 thì worker có tiếp tục spam không?
- Deploy nhiều instance thì cron có chạy trùng không?

### Reliability là gì?

> Là khả năng hệ thống tiếp tục cho kết quả đúng và phục hồi hợp lý khi có lỗi. Khi phỏng vấn em thích nói cụ thể các failure case ở trên hơn là chỉ nói “hệ thống phải reliable”.

---

# 9. Ví dụ: thiết kế Shopify synchronization service

## Bài nói

> Với dữ liệu Shopify mà application cần đọc thường xuyên, em lưu một bản cần thiết trong database local để tránh mỗi request đều phải gọi Shopify.
>
> Khi Shopify có thay đổi cần cập nhật nhanh, hệ thống nhận webhook. Webhook handler kiểm tra request, tránh xử lý trùng rồi nếu task dài thì đưa vào queue.
>
> Worker lấy job và update database, đồng thời giới hạn số job/request chạy cùng lúc để không làm quá tải database hoặc chạm Shopify rate limit.
>
> Vì webhook có thể bị miss hoặc xử lý lỗi, em giữ thêm một job định kỳ để lấy lại dữ liệu quan trọng và sửa những record bị lệch.
>
> Em theo dõi số webhook lỗi, số job đang chờ, thời gian từ lúc Shopify thay đổi tới lúc local DB cập nhật và số lần bị 429.

### Tại sao vừa webhook vừa cron?

> Webhook giúp cập nhật nhanh. Cron kiểm tra lại giúp sửa trường hợp webhook không tới hoặc xử lý thất bại.

### Nếu có 100k shops?

> Em không để một cron process cả 100k shop bằng `Promise.all`. Em chia workload thành job nhỏ trong queue, chạy nhiều worker nếu cần nhưng vẫn giới hạn concurrency và request rate theo khả năng DB/API.

---

# 10. Exactly-once / at-least-once — câu dễ bị hỏi mẹo

### At-least-once nghĩa là gì?

> Một message/job được đảm bảo cố gắng giao ít nhất một lần, nhưng có thể giao lại nhiều lần. Vì vậy consumer phải chịu được duplicate.

### Exactly-once nghĩa là gì?

> Theo cách nói đơn giản, business effect chỉ xảy ra đúng một lần. Trong distributed system, đạt “exactly-once” end-to-end rất khó vì network/retry có thể làm cùng message được xử lý lại.
>
> Vì vậy trong nhiều flow em thực tế hơn khi chấp nhận message có thể được giao lại và thiết kế processing idempotent để kết quả cuối không bị duplicate.

### Có nên chủ động nói “exactly-once” không?

> Nếu interviewer chưa hỏi, em thường không chủ động dùng từ này. Em nói cụ thể hơn: **“queue có thể giao job lại nên handler của em phải chịu được việc chạy hai lần.”** Cách nói này ít mơ hồ hơn.

---

# Checklist nói System Design

`Hệ thống cần gì → scale bao nhiêu → flow đơn giản → database → chỗ nào thật sự cần cache/queue → external failure → nếu component chết thì sao → đo bottleneck → mới scale tiếp`

## Câu kết tốt

> Đây là thiết kế mức tổng quan ban đầu. Em muốn dựa vào requirement và bottleneck thực tế để đi sâu tiếp, thay vì thêm nhiều component chỉ để architecture nhìn phức tạp hơn.
