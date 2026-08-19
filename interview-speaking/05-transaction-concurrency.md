# 05 — Transaction & Database Concurrency

# 1. Transaction

## Bài nói

> Em dùng transaction khi nhiều database operations phải được coi như một business operation duy nhất. Ví dụ tạo order và các records liên quan: nếu một bước fail thì không muốn database ở trạng thái nửa thành công nửa thất bại.
>
> Transaction giúp đảm bảo các property ACID, nhưng transaction càng dài thì càng giữ resource/lock lâu. Vì vậy em cố giữ transaction ngắn và tránh gọi external API bên trong transaction nếu không thực sự cần.

### ACID?

> Atomicity: all-or-nothing. Consistency: transaction đưa DB từ valid state sang valid state. Isolation: concurrent transactions hạn chế ảnh hưởng lẫn nhau theo isolation level. Durability: commit rồi thì dữ liệu phải được lưu bền vững theo guarantee của DB.

---

# 2. Race Condition

## Case: hai request cùng update

> Giả sử hai request cùng đọc stock = 1, cả hai đều nghĩ có thể bán rồi cùng update. Nếu chỉ read rồi write bằng application logic thì có thể oversell.
>
> Em sẽ cân nhắc atomic update có condition, transaction với locking hoặc optimistic concurrency tùy workload. Điểm quan trọng là correctness phải được enforce ở nơi có thể chịu concurrency, không chỉ bằng `if` trong Node.js.

### Optimistic vs pessimistic locking?

> Optimistic phù hợp khi conflict ít: dùng version/check khi update và retry khi conflict. Pessimistic lock row trước khi thay đổi, phù hợp khi cần serialize access nhưng có trade-off contention và deadlock.

---

# 3. Isolation Level

> Isolation level là trade-off giữa consistency và concurrency. Các anomaly thường được hỏi gồm dirty read, non-repeatable read và phantom read. Em không cố chọn level cao nhất cho mọi transaction; em chọn theo business invariant và database đang dùng.

---

# 4. Deadlock

## Bài nói

> Deadlock có thể xảy ra khi transaction A giữ resource 1 chờ resource 2, trong khi B giữ resource 2 chờ resource 1. Database thường detect và abort một transaction.
>
> Để giảm deadlock em giữ transaction ngắn, access resources theo thứ tự nhất quán, tạo index phù hợp để giảm phạm vi lock và retry transaction khi gặp deadlock nếu operation an toàn để retry.

---

# 5. External API + Database transaction

### Có nên mở transaction rồi gọi Stripe/Shopify?

> Thường em tránh vì network call có latency và có thể timeout, làm transaction giữ connection/lock lâu. Với distributed workflow em cân nhắc state machine, idempotency, outbox/event hoặc compensation thay vì cố biến external service thành một phần của local ACID transaction.

## Cách nhớ

`Atomic business operation → race condition → locking/version → short transaction → retry`
