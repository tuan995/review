# 13 — System Design Speaking Framework

Mục tiêu của chương này không phải học thuộc một architecture mẫu. Mục tiêu là **biết nói suy nghĩ theo từng bước**, giải thích được tại sao thêm một component và tránh dùng thuật ngữ lớn khi chưa cần.

---

# 1. Bắt đầu bằng requirements

## 💬 Bài nói

> Trước khi thiết kế em muốn xác nhận hệ thống cần làm gì, lượng traffic/data khoảng bao nhiêu và yêu cầu nào quan trọng nhất. Em không muốn thêm cache, queue hay sharding ngay khi chưa biết bottleneck thực tế.

## 🧾 Thuật ngữ

### **Functional requirement** *(hệ thống phải làm được chức năng gì)*

Ví dụ upload file, sync product, tạo payment.

### **Non-functional requirement** *(yêu cầu về chất lượng hệ thống)*

Ví dụ latency, availability, security, scalability.

### **Latency** *(thời gian từ khi request bắt đầu tới khi có response)*

### **Availability** *(khả năng hệ thống vẫn phục vụ được khi một phần gặp lỗi)*

### **Scalability** *(khả năng hệ thống tiếp tục xử lý được khi traffic/data tăng)*

### **Correctness** *(dữ liệu và business behavior vẫn đúng)*

⚠️ **Dễ bị bắt bẻ:**

> “Hệ thống cần high availability và high scalability.”

✅ **Cách nói an toàn:**

> Em hỏi cụ thể downtime có chấp nhận không, traffic hiện tại/dự kiến bao nhiêu và phần nào quan trọng nhất. Không phải mọi hệ thống đều cần cùng mức availability/scale.

---

# 2. Ước lượng scale

Không cần tính cực kỳ chính xác. Mục tiêu là biết order of magnitude.

Ví dụ interviewer nói:

- 100.000 shops;
- mỗi shop 10.000 products;
- mỗi product update vài lần/ngày.

Em có thể suy nghĩ:

- tổng số entity;
- request/event rate trung bình;
- peak traffic;
- storage growth;
- external API rate limit.

### **Peak traffic** *(mức tải cao nhất trong một khoảng thời gian, không phải chỉ average)*

### **Order of magnitude** *(ước lượng cỡ hàng nghìn, triệu, tỷ thay vì cần số chính xác)*

---

# 3. High-level design trước

## 💬 Bài nói

> Em bắt đầu từ flow đơn giản nhất: client → API → service → database. Sau đó em mới thêm Redis, queue hoặc search nếu requirement cho thấy cần.

```text
Client
  |
Load Balancer
  |
API instances
  |
  +---- Redis
  |
Database
  |
Queue → Workers → External APIs
```

### **High-level design** *(sơ đồ các component chính và cách chúng giao tiếp)*

### **Component** *(một phần của hệ thống như API, DB, queue, worker)*

⚠️ Không cần vẽ 15 component ngay từ phút đầu.

---

# 4. Load Balancer & Horizontal Scaling

## **Load balancer** *(phân phối request tới nhiều application instance)*

## **Horizontal scaling** *(tăng số instance/server thay vì chỉ tăng sức mạnh một server)*

## **Vertical scaling** *(tăng CPU/RAM cho server hiện tại)*

## 💬 Cách nói

> Nếu API không giữ state quan trọng chỉ trong memory của một instance thì em có thể scale nhiều instance sau load balancer dễ hơn.

### **Stateless API** *(một request không phụ thuộc vào session/state chỉ tồn tại trong memory của đúng một server)*

Ví dụ auth dùng token hoặc session lưu ở shared storage thay vì memory của instance A.

⚠️ “Stateless” không có nghĩa hệ thống không có database/state; nó chỉ nói application instance không giữ state riêng bắt buộc request sau phải quay lại đúng instance đó.

---

# 5. Chọn Database

## 💬 Bài nói

> Em chọn database dựa trên data model và access pattern. Nếu dữ liệu có relation rõ, cần transaction và constraint mạnh thì relational database như PostgreSQL thường là lựa chọn tốt. Nếu dữ liệu tự nhiên theo document và access pattern phù hợp thì có thể cân nhắc MongoDB.
>
> Em không chọn database chỉ vì nghe nói database đó scale tốt hơn.

### **Data model** *(cách mình tổ chức entity và relation)*

### **Access pattern** *(application thường đọc/ghi dữ liệu theo cách nào)*

📌 Ví dụ nếu query chủ yếu là “orders của shop X theo thời gian”, index/schema nên phục vụ pattern đó.

---

# 6. Index

## 💬 Cách nói

> Em thiết kế index từ query quan trọng, không index tất cả column. Em xem filter, join, sort rồi validate bằng execution plan.

### **Hot query** *(query chạy rất nhiều hoặc ảnh hưởng lớn tới latency/load)*

---

# 7. Cache

## Khi nào thêm Redis?

> Nếu có dữ liệu được đọc rất nhiều nhưng ít đổi, hoặc computation/external API call tốn kém, em cân nhắc cache.

Nhưng phải trả lời tiếp:

- TTL bao nhiêu?
- dữ liệu có thể stale bao lâu?
- invalidation thế nào?
- Redis down thì sao?

### **Cache invalidation** *(làm dữ liệu cache cũ mất hiệu lực khi source thay đổi)*

### **Stale data** *(cache chưa cập nhật theo dữ liệu mới)*

⚠️ **Dễ bị bắt bẻ:**

> “Thêm Redis để giảm latency.”

✅ **Cách nói an toàn:**

> Em chỉ thêm cache khi xác định read path hoặc external call là bottleneck và business chấp nhận được trade-off freshness/invalidation.

---

# 8. Queue

## **Queue** *(hàng đợi lưu task để worker xử lý sau)*

## 💬 Bài nói

> Em dùng queue khi task chạy lâu, cần retry, cần absorb traffic burst hoặc cần giới hạn tốc độ gọi downstream. Queue giúp request chính không phải chờ toàn bộ workflow.

### **Decouple** *(tách hai phần để không phải chạy đồng thời trong cùng request/process)*

Ví dụ API chỉ enqueue job; worker xử lý Shopify API sau.

### **Absorb burst** *(queue giữ lại lượng job tăng đột ngột để worker xử lý dần)*

⚠️ Queue không làm work biến mất; nếu producer nhanh hơn consumer lâu dài thì backlog vẫn tăng.

---

# 9. Worker scaling

## 💬 Cách nói

> Worker có thể scale nhiều instance, nhưng em không tăng worker vô hạn. Worker cuối cùng vẫn gọi database hoặc external API nên concurrency phải dựa trên capacity của downstream.

### **Downstream** *(hệ thống mà worker gọi tới phía sau, ví dụ DB/Shopify)*

### **Backlog** *(job đang chờ chưa xử lý)*

### **Throughput** *(số job xử lý được trong một đơn vị thời gian)*

---

# 10. External API

## 💬 Bài nói

> Với Shopify, Stripe hoặc Google, em coi external service là dependency có thể timeout, rate limit hoặc trả lỗi. Vì vậy em nghĩ trước retry, idempotency và trường hợp dữ liệu local bị lệch.

### **Dependency** *(hệ thống khác mà service của mình phụ thuộc vào)*

### **Idempotency** *(operation chạy lại nhưng không tạo kết quả sai hoặc nhân đôi)*

### **Reconciliation** *(job định kỳ kiểm tra lại hai hệ thống và sửa chênh lệch)*

---

# 11. Reliability

Khi thiết kế, tự hỏi:

- nếu worker chết giữa job?
- nếu webhook gửi hai lần?
- nếu DB update thành công nhưng external API fail?
- nếu Redis down?
- nếu API trả 429?
- nếu deploy 5 instance thì cron chạy mấy lần?

### **Failure mode** *(một cách cụ thể mà hệ thống có thể lỗi)*

System design tốt không chỉ mô tả happy path.

---

# 12. Replication

## **Replication** *(giữ nhiều bản copy dữ liệu trên nhiều database node)*

Có thể dùng để:

- tăng availability;
- phục vụ read ở replica;
- hỗ trợ failover.

### **Read replica** *(database replica chủ yếu phục vụ đọc)*

⚠️ Replica có thể có replication lag.

### **Replication lag** *(replica chậm hơn primary một khoảng thời gian)*

Vì vậy request vừa write xong rồi read từ replica có thể chưa thấy dữ liệu mới.

---

# 13. Partitioning / Sharding

## **Partitioning** *(chia dữ liệu thành nhiều phần)*

## **Sharding** *(chia dữ liệu qua nhiều database/server dựa trên shard key)*

## 💬 Cách nói

> Em không nhảy tới sharding ngay. Trước tiên em tối ưu query/index, kiểm soát connection và xem vertical scaling/read replica có đủ không. Sharding chỉ nên thêm khi một database/node không còn đáp ứng requirement hoặc có lý do rõ ràng.

### **Shard key** *(field/quy tắc quyết định một record nằm ở shard nào)*

📌 Ví dụ `shop_id` có thể là candidate nếu workload được tách tự nhiên theo shop, nhưng cần xem distribution và query cross-shop.

### **Hot shard** *(một shard nhận quá nhiều traffic so với shard khác)*

---

# 14. Consistency

## Nói đơn giản trước

> Nếu dữ liệu được lưu nhiều nơi, có thể các bản copy không cập nhật cùng đúng một thời điểm. Em cần biết nghiệp vụ nào bắt buộc đọc dữ liệu mới nhất và nghiệp vụ nào chấp nhận chậm vài giây/phút.

### **Strong consistency** *(sau write, read theo guarantee phù hợp thấy state mới nhất)*

### **Eventual consistency** *(các bản copy có thể lệch tạm thời nhưng sẽ hội tụ lại sau)*

⚠️ Đây là khái niệm lớn. Khi phỏng vấn nên gắn với ví dụ thay vì chỉ nói tên.

📌 Inventory checkout cần correctness cao hơn analytics dashboard.

---

# 15. At-least-once vs Exactly-once

## **At-least-once** *(message/job có thể được giao lại nhiều lần để giảm nguy cơ mất)*

## 💬 Cách nói an toàn

> Với queue/webhook em thường giả định event có thể tới hơn một lần, nên consumer phải idempotent.

### **Exactly-once** *(mục tiêu mỗi logical event chỉ tạo effect đúng một lần)*

Trong distributed system đây là chủ đề nhiều nuance. Không nên nói:

> “Queue của em đảm bảo exactly-once nên không cần idempotency.”

✅ Nói:

> Em ưu tiên thiết kế idempotent processing để chịu được duplicate delivery.

---

# 16. Observability

## **Observability** *(khả năng hiểu hệ thống đang xảy ra gì qua logs, metrics, traces)*

Một design nên nghĩ tới:

- request latency/error;
- DB usage;
- queue depth;
- worker failure;
- external API 429;
- sync lag;
- business mismatch.

### **Sync lag** *(thời gian dữ liệu local chậm hơn source)*

---

# 17. Security

Những câu hỏi cơ bản:

- authentication?
- authorization?
- secret lưu ở đâu?
- data encryption?
- validation?
- webhook signature?
- rate limiting?
- audit log nếu cần?

### **Authentication** *(ai đang gọi)*

### **Authorization** *(người đó được phép làm gì)*

---

# 18. Ví dụ: Shopify Synchronization Service

## 💬 Bài nói 2 phút

> Em coi Shopify là nguồn dữ liệu chính cho product/inventory thuộc Shopify. Application cần đọc dữ liệu thường xuyên nên em lưu một bản cần thiết trong database local để giảm latency và số API call.
>
> Với thay đổi cần cập nhật nhanh, Shopify gửi webhook tới backend. Webhook handler verify request, kiểm tra duplicate rồi nếu xử lý dài thì enqueue job.
>
> Worker lấy job và update database. Em giới hạn worker concurrency và request rate vì cả database lẫn Shopify API đều có capacity/limit.
>
> Vì webhook có thể delay hoặc bị miss, em có scheduled reconciliation job để định kỳ đọc lại dữ liệu quan trọng từ Shopify và sửa record bị lệch.
>
> Nếu scale lên nhiều shop, em chia workload thành job nhỏ theo shop/product thay vì một cron loop toàn bộ. Worker có thể scale horizontally nhưng vẫn giữ limiter theo downstream.
>
> Em theo dõi webhook failure, queue depth, oldest job age, sync lag, Shopify 429 và reconciliation mismatch để biết dữ liệu có đang bị chậm hay lệch không.

## 📌 Sơ đồ

```text
Shopify
   |
 webhook
   v
API endpoint
   |
verify + dedup
   |
Queue
   |
Workers ---- limiter ---- Shopify API
   |
Database
   ^
   |
Reconciliation cron
```

---

# 🎯 Interviewer hỏi tiếp

### Tại sao vừa webhook vừa cron?

> Webhook giúp cập nhật nhanh. Cron reconciliation là lớp kiểm tra lại nếu webhook bị miss hoặc processing fail.

### Nếu có 100k shops?

> Em không chạy một loop khổng lồ trong một process. Em partition workload thành jobs, queue chúng và scale workers. Nhưng em vẫn giới hạn concurrency/rate theo database và Shopify capacity.

### Nếu một shop có dữ liệu cực lớn?

> Em có thể chia tiếp theo page/product batch, checkpoint progress và tránh một job quá lớn giữ worker quá lâu.

### Nếu database down?

> Worker không nên ACK job thành công. Job có thể retry theo policy, nhưng cần backoff để không tạo thundering herd khi DB vừa hồi phục.

### Nếu Redis/queue down?

> API cần quyết định fail request tạo job hay persist theo một path khác tùy mức quan trọng. Em không nói queue “không bao giờ down”; availability của queue là một dependency cần thiết kế.

### CAP theorem có cần nhắc không?

> Chỉ khi interviewer hỏi distributed consistency sâu. Em không chủ động ném CAP vào mọi design. Em ưu tiên nói requirement cụ thể: khi network partition thì hệ thống ưu tiên availability hay consistency ở flow nào.

---

# 19. Framework nói System Design

Có thể đi theo thứ tự:

1. **Clarify requirement** — cần làm gì?
2. **Estimate scale** — traffic/data cỡ nào?
3. **API/high-level flow** — request đi đâu?
4. **Data model/DB** — dữ liệu lưu thế nào?
5. **Index** — query chính là gì?
6. **Cache** — có bottleneck read không?
7. **Queue/worker** — task nào async?
8. **External dependency** — timeout/rate limit?
9. **Failure cases** — component chết thì sao?
10. **Observability/security** — biết lỗi và bảo vệ hệ thống thế nào?
11. **Bottleneck/trade-off** — phần nào cần scale tiếp?

---

# ⚠️ Những câu dễ bị bắt bẻ

❌ “Em dùng microservice để scale.”  
✅ “Em chỉ tách service khi domain/deployment/scale cần độc lập; monolith vẫn có thể horizontal scale.”

❌ “NoSQL scale tốt hơn SQL.”  
✅ “Khả năng scale phụ thuộc engine/architecture; em chọn database trước hết theo data model, query và consistency need.”

❌ “Queue giải quyết traffic spike.”  
✅ “Queue absorb burst tạm thời; nếu input rate luôn cao hơn processing rate thì backlog vẫn tăng.”

❌ “Read replica tăng write performance.”  
✅ “Read replica chủ yếu offload read; write thường vẫn đi primary theo architecture phổ biến.”

❌ “Sharding để hệ thống scale.”  
✅ “Sharding là bước phức tạp; em chỉ thêm khi bottleneck/data scale thực sự yêu cầu.”

---

# 📌 Câu kết tốt

> Đây là high-level design ban đầu. Em sẽ đo bottleneck và đi sâu vào phần có requirement khó nhất thay vì thêm component chỉ để architecture trông phức tạp.

# Cách nhớ

**Requirement → scale → flow → DB/index → cache → queue/worker → external API → failure → observability/security → bottleneck/trade-off**