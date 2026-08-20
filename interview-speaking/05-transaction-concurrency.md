# 05 — Transaction & Database Concurrency

Mục tiêu: hiểu transaction bằng **nghiệp vụ cụ thể**, không trả lời bằng một chuỗi ACID/locking khó nhớ.

---

# 1. Transaction là gì?

## 💬 Bài nói

> Em dùng **transaction** khi nhiều thao tác database thuộc cùng một nghiệp vụ và em không muốn hệ thống chỉ làm thành công một nửa.
>
> Ví dụ khi tạo order, em có thể cần tạo `Order`, tạo các `LineItem` và cập nhật một số dữ liệu liên quan. Nếu tạo Order thành công nhưng tạo LineItem thất bại thì database bị trạng thái làm dở.
>
> Khi đặt các thao tác đó trong transaction, nếu tất cả thành công thì em **commit** — tức là xác nhận lưu thay đổi. Nếu một bước quan trọng thất bại thì em **rollback** — tức là hủy những thay đổi chưa commit của transaction đó.
>
> Em cũng cố giữ transaction ngắn vì trong thời gian transaction mở, nó giữ database connection và có thể giữ lock. Vì vậy em thường tránh gọi Stripe/Shopify trong lúc transaction đang mở.

---

# 🧾 Thuật ngữ

### **Transaction** *(nhóm thao tác database được xử lý như một đơn vị logic)*

### **Commit** *(xác nhận transaction thành công và lưu thay đổi)*

### **Rollback** *(hủy thay đổi chưa commit của transaction)*

### **Lock** *(cơ chế database kiểm soát các transaction cùng truy cập dữ liệu có xung đột)*

---

# 2. ACID — hiểu bằng ví dụ

## **Atomicity** *(hoặc tất cả thành công, hoặc không để lại trạng thái nửa chừng)*

📌 Tạo Order + 3 LineItem. Nếu LineItem thứ 2 fail thì transaction rollback thay vì để lại một order thiếu dữ liệu.

## **Consistency** *(sau transaction, dữ liệu vẫn thỏa rule/constraint của hệ thống)*

📌 Ví dụ `email` phải unique, foreign key phải hợp lệ, hoặc business rule không cho stock âm.

⚠️ **Không nên chỉ nói:**

> “Consistency là từ valid state sang valid state.”

Vì interviewer rất dễ hỏi “valid state là gì?”.

✅ **Cách nói tốt hơn:**

> Em hiểu consistency là dữ liệu sau transaction vẫn phải thỏa các constraint và business rule. Ví dụ unique email vẫn được giữ hoặc tồn kho không bị âm.

### **Valid state** *(trạng thái dữ liệu vẫn thỏa các rule đã định)*

Nếu dùng từ này thì giải thích luôn, nhưng tốt nhất nói rule cụ thể.

## **Isolation** *(transaction chạy đồng thời nhìn thấy và ảnh hưởng nhau ở mức nào)*

📌 Hai request cùng sửa một product. Isolation level quyết định một transaction thấy thay đổi của transaction kia khi nào và database bảo vệ các trường hợp concurrent ra sao.

## **Durability** *(database đã báo commit thành công thì dữ liệu phải được lưu bền vững theo guarantee của DB)*

Không có nghĩa application process không bao giờ crash; nó nói về guarantee của database sau commit.

---

# 3. Race Condition

## **Race condition** *(kết quả phụ thuộc vào thứ tự/thời điểm nhiều request chạy đồng thời)*

## 📌 Ví dụ stock = 1

```text
Request A đọc stock = 1
Request B đọc stock = 1
A kiểm tra > 0 → cho mua
B kiểm tra > 0 → cũng cho mua
A update
B update
→ có thể oversell
```

## 💬 Bài nói

> Vấn đề là đoạn “đọc → kiểm tra → ghi” gồm nhiều bước riêng. `if (stock > 0)` trong Node.js không đủ bảo vệ vì request khác có thể thay đổi dữ liệu sau lúc mình đọc nhưng trước lúc mình ghi.
>
> Em cần đẩy phần correctness xuống database bằng một thao tác atomic, transaction + locking, hoặc optimistic concurrency tùy workload.

### **Atomic operation** *(thao tác được thực hiện như một đơn vị không bị chen giữa ở điểm cần bảo vệ)*

Ví dụ:

```sql
UPDATE products
SET stock = stock - 1
WHERE id = ? AND stock > 0;
```

Câu query vừa kiểm tra điều kiện vừa update trong một database operation, an toàn hơn việc đọc stock ra Node rồi update sau.

---

# 4. Optimistic vs Pessimistic Locking

## **Optimistic locking** *(giả định conflict hiếm, phát hiện conflict lúc update)*

📌 Record có `version = 5`.

- A và B cùng đọc version 5.
- A update thành công và tăng version thành 6.
- B update với điều kiện `version = 5` thì không match.
- B biết dữ liệu đã bị thay đổi và có thể đọc lại/retry.

## **Pessimistic locking** *(lock dữ liệu trước vì giả định conflict có thể xảy ra và muốn transaction khác chờ)*

Ví dụ row lock khi một transaction đang xử lý stock nhạy cảm.

### Chọn cái nào?

> Nếu conflict ít, optimistic thường cho concurrency tốt hơn vì không giữ lock lâu. Nếu conflict cao hoặc nghiệp vụ rất nhạy cảm thì row locking có thể phù hợp hơn. Em chọn theo workload và database cụ thể.

### **Conflict** *(hai transaction muốn thay đổi dữ liệu theo cách không thể cùng đúng)*

---

# 5. Isolation Level

## **Isolation level** *(mức độ database tách các transaction chạy đồng thời khỏi nhau)*

Mức isolation cao hơn thường giảm một số anomaly nhưng có thể đổi lại bằng nhiều waiting/locking hơn tùy database.

### **Dirty read** *(đọc dữ liệu transaction khác chưa commit)*

A update giá trị nhưng chưa commit. B đọc được giá trị đó. Sau đó A rollback. B đã từng đọc một giá trị cuối cùng không tồn tại.

### **Non-repeatable read** *(đọc cùng một row hai lần trong transaction nhưng giá trị thay đổi vì transaction khác commit ở giữa)*

### **Phantom read** *(chạy cùng query điều kiện hai lần nhưng tập row thay đổi vì transaction khác insert/delete rồi commit)*

⚠️ **Dễ bị bắt bẻ:**

> “Isolation càng cao càng tốt.”

✅ **Cách nói an toàn:**

> Isolation cao hơn bảo vệ nhiều hiện tượng concurrency hơn nhưng có thể giảm concurrency hoặc tăng waiting. Em chọn dựa vào yêu cầu correctness của nghiệp vụ.

---

# 6. Deadlock

## **Deadlock** *(hai transaction giữ resource và chờ lẫn nhau nên không bên nào tự đi tiếp)*

```text
Transaction A: lock Order 1 → chờ Order 2
Transaction B: lock Order 2 → chờ Order 1
```

Database thường phát hiện vòng chờ và abort một transaction.

## 💬 Cách giảm deadlock

> Em giữ transaction ngắn, truy cập resource theo thứ tự nhất quán khi có thể, tránh query scan/lock phạm vi quá lớn và xử lý retry nếu database abort transaction vì deadlock và operation an toàn để chạy lại.

### **Abort transaction** *(database dừng transaction và rollback để phá tình trạng lỗi như deadlock)*

---

# 7. Tại sao không gọi external API trong DB transaction?

## 💬 Bài nói

> Stripe hoặc Shopify là network call nên latency khó đoán và có thể timeout. Nếu em mở transaction rồi giữ connection/lock trong lúc chờ external API vài giây, database resource cũng bị giữ vài giây.
>
> Khi nhiều request cùng làm như vậy, connection pool có thể hết và lock contention tăng. Vì vậy local DB transaction chỉ nên bảo vệ phần dữ liệu trong database của mình. External workflow thường được chia thành các bước có state rõ ràng và xử lý retry/idempotency.

### **State** *(trạng thái hiện tại của một workflow)*

Ví dụ payment có thể là `pending`, `paid`, `failed`.

### **Idempotency** *(chạy lại cùng operation nhưng không tạo kết quả sai hoặc nhân đôi)*

📌 Stripe gửi lại cùng webhook hai lần nhưng hệ thống không tạo hai payment record.

---

# 8. Compensation

## **Compensation** *(hành động nghiệp vụ để bù lại một bước đã thành công khi bước sau thất bại)*

📌 Ví dụ:

1. Đã charge payment thành công.
2. Bước tạo order sau đó thất bại nghiêm trọng.
3. Không thể dùng DB rollback để “xóa” payment ở Stripe.
4. Có thể cần một hành động hoàn tiền hoặc hủy phù hợp.

⚠️ Compensation không phải rollback kỹ thuật của distributed system; nó là hành động nghiệp vụ bù lại.

---

# 9. Câu hỏi đào sâu

### Tại sao `if` trong application không đủ chống race condition?

> Vì check và update là hai thời điểm khác nhau. Request khác có thể thay đổi dữ liệu ở giữa.

### Transaction có đảm bảo không bao giờ duplicate không?

> Không tự động. Transaction chỉ giúp bảo vệ các thao tác DB theo guarantee của database. Duplicate còn phụ thuộc constraint, idempotency/business key và flow cụ thể.

### Unique constraint có giúp concurrency không?

> Có thể. Ví dụ nếu chỉ được có một record cho một business key, unique constraint giúp database enforce rule ngay cả khi hai request cùng insert.

### Connection pool liên quan transaction thế nào?

> Mỗi transaction thường giữ một database connection trong thời gian transaction chạy. Transaction quá dài làm connection được trả về pool chậm hơn.

---

# ⚠️ Những câu dễ bị hỏi mẹo

❌ “Transaction đảm bảo consistency.”  
✅ “Transaction kết hợp với constraint và logic giúp dữ liệu vẫn thỏa các rule; em sẽ nói rule cụ thể của nghiệp vụ.”

❌ “Pessimistic locking luôn an toàn hơn.”  
✅ “Nó serialize access mạnh hơn nhưng có thể tăng waiting/deadlock; em chọn theo conflict rate.”

❌ “Serializable là tốt nhất.”  
✅ “Serializable bảo vệ mạnh nhưng cost cao hơn; không phải mọi nghiệp vụ đều cần.”

---

# 📌 Cách nhớ

**Nhiều bước của một nghiệp vụ → transaction → commit/rollback → nhiều request cùng sửa → race condition → atomic update/locking/version → transaction ngắn → external API dùng state + retry + idempotency**