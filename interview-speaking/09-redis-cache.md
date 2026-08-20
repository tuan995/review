# 09 — Redis & Caching

Mục tiêu: hiểu Redis/cache từ **vấn đề cần giải quyết**, không trả lời theo kiểu “Redis nhanh nên dùng Redis”.

---

# 1. Redis là gì và khi nào dùng?

## 💬 Bài nói

> Redis là một in-memory data store, tức là dữ liệu chủ yếu được truy cập từ memory nên tốc độ rất nhanh. Em thường dùng Redis cho những dữ liệu tạm hoặc cần truy cập nhanh như cache, temporary state, rate limiting hoặc một số coordination use case.
>
> Tuy nhiên em không thêm Redis chỉ vì nó nhanh. Em xem bottleneck có thật sự nằm ở database/external API không, dữ liệu có chấp nhận cũ trong một khoảng thời gian không và nếu Redis bị lỗi thì hệ thống nên phản ứng thế nào.

---

# 🧾 Thuật ngữ

### **In-memory** *(dữ liệu chủ yếu nằm trong RAM để truy cập nhanh)*

### **Cache** *(bản dữ liệu tạm giúp tránh phải đọc nguồn chậm hơn nhiều lần)*

### **TTL — Time To Live** *(thời gian key còn tồn tại trước khi tự hết hạn)*

### **Bottleneck** *(thành phần đang giới hạn performance của toàn flow)*

---

# 2. Cache-aside

## **Cache-aside** *(application tự đọc cache trước; nếu không có thì đọc DB rồi ghi lại cache)*

## 📌 Flow

```text
Request
   ↓
Redis
 ├─ hit → trả dữ liệu
 └─ miss
      ↓
   Database
      ↓
   set Redis
      ↓
   trả dữ liệu
```

## 💬 Bài nói

> Pattern em thường dùng là cache-aside. Application đọc Redis trước. Nếu có dữ liệu thì trả luôn. Nếu không có thì đọc database, sau đó lưu kết quả vào Redis với TTL để lần sau đọc nhanh hơn.
>
> Khi dữ liệu gốc thay đổi, em cần quyết định update cache hay xóa cache để lần đọc sau rebuild lại.

### **Cache hit** *(tìm thấy dữ liệu trong cache)*

### **Cache miss** *(không tìm thấy dữ liệu trong cache nên phải đọc nguồn gốc)*

---

# 3. Cache invalidation

## **Cache invalidation** *(làm cache cũ mất hiệu lực khi dữ liệu gốc thay đổi)*

Có vài cách phổ biến:

- xóa key sau khi database update;
- update lại cache;
- dùng TTL để cache tự hết hạn;
- kết hợp nhiều cách tùy consistency requirement.

## 💬 Cách nói

> Phần khó của cache không phải đọc nhanh mà là giữ cache không sai quá lâu. Nếu business cần dữ liệu khá mới, em thường invalidation khi write và vẫn có TTL như một safety net.

### **Safety net** *(lớp bảo vệ phụ nếu cơ chế chính bị miss)*

---

# 4. Stale data

## **Stale data** *(cache đang giữ giá trị cũ hơn dữ liệu gốc)*

📌 Ví dụ database stock = 3 nhưng Redis vẫn trả 5.

## 💬 Cách nói

> Trước khi cache em phải biết business có chấp nhận dữ liệu cũ trong bao lâu. Product description có thể chấp nhận chậm vài phút, nhưng stock hoặc permission nhạy cảm có thể không phù hợp để cache lâu.

⚠️ **Dễ bị bắt bẻ:**

> “Redis cache luôn consistent với DB.”

✅ **Cách nói an toàn:**

> Cache có thể stale. Em thiết kế TTL/invalidation theo mức freshness business chấp nhận.

---

# 5. Cache stampede

## Nói hiện tượng trước

> Giả sử một key rất hot vừa hết hạn. Cùng lúc 500 request đều cache miss và cả 500 request cùng query database để rebuild cùng một dữ liệu. Database có thể nhận một burst lớn.

Hiện tượng này thường gọi là **cache stampede**.

## Cách giảm

- lock/single-flight để một request rebuild, request khác chờ hoặc dùng dữ liệu cũ;
- **stale-while-revalidate** *(tạm trả dữ liệu cũ trong lúc background refresh)*;
- randomize TTL để nhiều key không hết hạn cùng lúc.

### **Single-flight** *(nhiều request cùng cần một dữ liệu nhưng chỉ một operation thực sự fetch/rebuild)*

---

# 6. Chọn TTL bao nhiêu?

## 💬 Cách nói

> Không có TTL đúng cho mọi dữ liệu. Em chọn dựa trên business freshness, tần suất đọc, tần suất thay đổi và cost khi rebuild.

📌 Ví dụ:

- config ít đổi → TTL dài hơn;
- inventory thay đổi liên tục → TTL ngắn hoặc không cache theo cách đơn giản;
- external API rất đắt → cache lâu hơn nếu business cho phép.

⚠️ Nếu nói “TTL 5 phút” hãy sẵn sàng trả lời “tại sao 5 phút?”.

---

# 7. Redis down thì sao?

## 💬 Bài nói

> Nếu Redis chỉ là cache, em thường muốn application vẫn có thể fallback về database. Nhưng phải cẩn thận vì nếu Redis down, toàn bộ traffic có thể đổ xuống DB và làm DB quá tải.
>
> Nếu Redis đang giữ state quan trọng chứ không chỉ cache, requirement về persistence, replication và high availability sẽ khác nhiều.

### **Fallback** *(dùng phương án thay thế khi dependency chính không dùng được)*

### **High Availability / HA** *(thiết kế để service vẫn phục vụ khi một node gặp lỗi)*

---

# 8. Redis có phải database chính không?

Có thể trong một số architecture, nhưng không nên mặc định.

## 💬 Cách nói an toàn

> Trong các case cache của em, database chính vẫn là source of truth và Redis là lớp tăng tốc. Nếu dùng Redis làm state chính thì phải thiết kế persistence/replication/failure recovery riêng.

---

# 9. Eviction

## **Eviction** *(Redis tự loại key khi memory đầy theo policy cấu hình)*

Nếu Redis có memory limit, khi đầy nó có thể:

- reject write;
- hoặc xóa key theo eviction policy.

⚠️ Đây là lý do không nên coi cache key chắc chắn tồn tại cho đến TTL.

---

# 10. Rate limiting bằng Redis

Redis thường được dùng để lưu counter/token vì update nhanh và có TTL.

📌 Ví dụ:

```text
user:123:requests:minute = 37
TTL = thời gian còn lại của cửa sổ
```

Nhưng thuật toán rate limit có nhiều loại: fixed window, sliding window, token bucket... Nếu chưa cần thì không mở quá sâu.

---

# 🎯 Interviewer hỏi tiếp

### Tại sao Redis nhanh?

> Dữ liệu chủ yếu truy cập trong memory, cấu trúc dữ liệu được tối ưu và Redis xử lý command rất hiệu quả. Nhưng network round-trip vẫn tồn tại, nên không phải “0 latency”.

### Cache có thể làm data sai không?

> Có nếu invalidation sai hoặc TTL quá dài. Vì vậy cache không nên là nơi duy nhất enforce business invariant quan trọng nếu source of truth nằm ở DB.

### Nếu delete cache trước rồi DB update fail?

> Cache miss sẽ đọc lại DB cũ và rebuild lại giá trị cũ. Flow write/invalidation cần được thiết kế theo consistency requirement; không có một thứ tự đúng cho mọi hệ thống.

### Redis lock có luôn an toàn không?

> Không nên nói tuyệt đối. Distributed lock cần ownership, TTL và xử lý expiry/process pause. Với nghiệp vụ rất nhạy cảm em phải xem guarantee thực tế thay vì coi một `SETNX` đơn giản là đủ.

---

# ⚠️ Những câu dễ bị bắt bẻ

❌ “Redis nhanh hơn DB nên em cache mọi thứ.”  
✅ “Em cache khi read cost cao và business chấp nhận được trade-off freshness/invalidation.”

❌ “TTL giải quyết consistency.”  
✅ “TTL giới hạn thời gian cache tồn tại nhưng trong thời gian đó dữ liệu vẫn có thể stale.”

❌ “Redis down thì cứ fallback DB.”  
✅ “Có thể fallback nếu Redis chỉ là cache, nhưng cần bảo vệ DB khỏi traffic spike.”

---

# 📌 Cách nhớ

**Có bottleneck không? → cache-aside → hit/miss → TTL → invalidation → stale data → stampede → Redis down/fallback**