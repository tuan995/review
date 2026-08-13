# Transaction & ACID

## 1. Transaction là gì?

Transaction là một nhóm thao tác database được xem như **một đơn vị công việc duy nhất**.

Hoặc tất cả thao tác thành công và được `COMMIT`, hoặc khi có lỗi thì các thay đổi được `ROLLBACK`.

Ví dụ chuyển 100.000đ từ A sang B:

```sql
BEGIN;

UPDATE accounts
SET balance = balance - 100000
WHERE id = 'A';

UPDATE accounts
SET balance = balance + 100000
WHERE id = 'B';

COMMIT;
```

Nếu bước cộng tiền cho B thất bại, ta không muốn A vẫn bị trừ tiền. Transaction giải quyết vấn đề đó.

---

## 2. ACID là gì?

ACID gồm 4 tính chất quan trọng của transaction:

- **A — Atomicity**: tất cả hoặc không có gì.
- **C — Consistency**: dữ liệu phải chuyển từ trạng thái hợp lệ này sang trạng thái hợp lệ khác.
- **I — Isolation**: các transaction chạy đồng thời không được gây ra kết quả sai do nhìn thấy trạng thái trung gian không phù hợp.
- **D — Durability**: sau khi commit thành công, dữ liệu phải được lưu bền vững ngay cả khi hệ thống gặp sự cố.

### Mẹo nhớ

> Atomicity = All or Nothing  
> Consistency = Rules remain valid  
> Isolation = Transactions don't interfere incorrectly  
> Durability = Commit means it stays

---

## 3. Atomicity

Giả sử transaction có 3 bước:

```text
1. Trừ tiền A
2. Cộng tiền B
3. Ghi transaction history
```

Nếu bước 3 lỗi thì toàn bộ transaction có thể rollback.

Không được để database ở trạng thái chỉ thực hiện một nửa nghiệp vụ.

---

## 4. Consistency

Consistency liên quan đến các invariant/rule của hệ thống.

Ví dụ:

- foreign key phải hợp lệ
- unique constraint không được vi phạm
- balance không được âm nếu business rule cấm
- tổng tiền trước và sau transfer phải đúng theo nghiệp vụ

Lưu ý: database constraints giúp bảo vệ consistency, nhưng application cũng phải thực thi đúng business rules.

---

## 5. Isolation

Nhiều transaction thường chạy đồng thời.

Ví dụ hai request cùng đọc stock = 1 rồi cùng mua sản phẩm cuối cùng. Nếu concurrency không được xử lý đúng, cả hai có thể nghĩ rằng mình mua thành công.

Isolation level và locking/MVCC giúp kiểm soát những tình huống này.

Chi tiết: xem `isolation-lock-deadlock.md`.

---

## 6. Durability

Sau khi database báo `COMMIT` thành công, dữ liệu cần tồn tại kể cả khi process/database server crash.

Database thường sử dụng cơ chế như WAL (Write-Ahead Log), redo log và flush xuống storage để hỗ trợ durability.

---

## 7. COMMIT và ROLLBACK

```sql
BEGIN;

UPDATE users
SET balance = balance - 100
WHERE id = 1;

COMMIT;
```

Nếu có lỗi:

```sql
BEGIN;

UPDATE users
SET balance = balance - 100
WHERE id = 1;

ROLLBACK;
```

`COMMIT` xác nhận transaction. `ROLLBACK` hủy các thay đổi chưa commit của transaction.

---

## 8. Transaction có làm hệ thống chậm không?

Có thể.

Transaction giữ tài nguyên lâu có thể gây:

- lock contention
- transaction khác phải chờ
- deadlock
- tăng version/undo data tùy database
- giảm throughput

Vì vậy nên giữ transaction **ngắn nhất có thể**.

Không nên mở transaction rồi gọi API bên ngoài mất vài giây nếu không thực sự cần thiết.

---

## 9. Transaction trong backend thực tế

Ví dụ tạo order:

```text
BEGIN
  create order
  create order_items
  decrease inventory
  create payment record
COMMIT
```

Nếu một bước thất bại, rollback để tránh dữ liệu nửa vời.

Nhưng nếu nghiệp vụ trải qua **nhiều service/database khác nhau**, một local database transaction không thể tự đảm bảo atomicity toàn hệ thống. Khi đó có thể cần Saga, Outbox, idempotency hoặc các pattern distributed transaction khác.

---

# Câu hỏi phỏng vấn

## Q1. Transaction là gì?

Một nhóm database operations được xử lý như một đơn vị logic. Các thao tác được commit cùng nhau hoặc rollback khi thất bại.

## Q2. ACID là gì?

Atomicity, Consistency, Isolation, Durability.

## Q3. Atomicity khác Consistency thế nào?

Atomicity nói về **all-or-nothing của transaction**. Consistency nói về việc dữ liệu vẫn tuân thủ các constraint/invariant sau transaction.

## Q4. Commit là gì?

Xác nhận transaction thành công và làm các thay đổi trở thành trạng thái đã commit.

## Q5. Rollback là gì?

Hủy các thay đổi chưa commit của transaction và đưa dữ liệu về trạng thái phù hợp trước transaction.

## Q6. Vì sao transaction dài nguy hiểm?

Vì nó có thể giữ lock/snapshot/resource lâu, tăng contention và xác suất deadlock, từ đó làm giảm throughput.

## Q7. Có nên gọi HTTP API bên ngoài bên trong DB transaction không?

Thường nên tránh vì network call có latency và có thể timeout, khiến transaction giữ tài nguyên quá lâu. Tùy nghiệp vụ có thể dùng Outbox/Saga hoặc tách workflow.

## Q8. Transaction có giải quyết distributed transaction giữa nhiều microservice không?

Không. Local DB transaction chỉ bảo vệ phạm vi database/transaction manager tương ứng. Distributed workflow thường cần Saga, Outbox, idempotency hoặc giải pháp coordination khác.

---

# Cheat Sheet

```text
Transaction = một đơn vị công việc

BEGIN
  operations
COMMIT

error -> ROLLBACK

ACID
A = Atomicity    -> all or nothing
C = Consistency  -> rules/invariants remain valid
I = Isolation    -> concurrent transactions controlled
D = Durability   -> committed data survives failures

Best practice:
- transaction càng ngắn càng tốt
- tránh network call dài bên trong transaction
- hiểu isolation level
- chuẩn bị retry khi deadlock/serialization failure
```
