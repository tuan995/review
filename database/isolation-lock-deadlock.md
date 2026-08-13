# Isolation Level, Lock & Deadlock

## 1. Vì sao cần Isolation?

Khi nhiều transaction chạy đồng thời, chúng có thể đọc/ghi cùng dữ liệu và tạo ra race condition hoặc kết quả không mong muốn.

Ví dụ stock = 1:

```text
Transaction A: đọc stock = 1
Transaction B: đọc stock = 1
Transaction A: mua 1 sản phẩm
Transaction B: cũng mua 1 sản phẩm
```

Nếu không kiểm soát concurrency đúng cách, hệ thống có thể oversell.

---

# 2. Các anomaly quan trọng

## Dirty Read

Transaction A đọc dữ liệu mà Transaction B **chưa commit**.

```text
B: balance 100 -> 0
A: đọc balance = 0
B: ROLLBACK -> balance trở lại 100
```

A đã sử dụng một giá trị chưa từng thực sự được commit.

## Non-repeatable Read

Trong cùng một transaction, đọc cùng một row hai lần nhưng nhận hai giá trị khác nhau vì transaction khác đã update và commit ở giữa.

```text
A: SELECT balance -> 100
B: UPDATE balance -> 200; COMMIT
A: SELECT balance -> 200
```

## Phantom Read

Cùng một query theo điều kiện chạy hai lần nhưng số row/kết quả tập row thay đổi do transaction khác insert/delete row phù hợp với điều kiện.

```text
A: SELECT * FROM orders WHERE status = 'pending'; -- 10 rows
B: INSERT pending order; COMMIT
A: SELECT ...                                     -- 11 rows
```

## Lost Update

Hai transaction cùng đọc một giá trị rồi update dựa trên giá trị cũ, khiến update của một transaction bị ghi đè.

```text
counter = 10
A reads 10
B reads 10
A writes 11
B writes 11
```

Kết quả là 11 thay vì 12.

---

# 3. Isolation Levels

SQL standard thường nói đến 4 mức:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

Isolation càng mạnh thường càng dễ reasoning về correctness, nhưng có thể tăng blocking, abort/retry hoặc giảm concurrency tùy implementation.

> Hành vi chính xác phụ thuộc database và cơ chế MVCC/locking của nó. Không nên học bảng isolation như một quy luật tuyệt đối cho mọi DB engine.

## READ UNCOMMITTED

Isolation thấp nhất theo SQL standard. Có thể cho phép dirty read.

Ít được sử dụng trong application thông thường.

## READ COMMITTED

Mỗi statement chỉ nhìn thấy dữ liệu đã commit theo semantics của database.

Thông thường ngăn dirty read, nhưng một transaction có thể thấy giá trị khác nhau giữa hai lần đọc.

PostgreSQL mặc định dùng `READ COMMITTED`.

## REPEATABLE READ

Mục tiêu là những row đã đọc không thay đổi theo cách tạo non-repeatable read trong transaction.

Với MVCC, transaction thường đọc từ một snapshot nhất quán, nhưng chi tiết phantom/write-conflict phụ thuộc database.

MySQL InnoDB mặc định dùng `REPEATABLE READ`.

## SERIALIZABLE

Mức isolation mạnh nhất trong SQL standard.

Kết quả phải tương đương với một thứ tự chạy tuần tự hợp lệ của các transaction.

Database có thể đạt điều này bằng locking hoặc phát hiện conflict và abort transaction. Vì vậy application phải sẵn sàng retry serialization failures.

---

# 4. MVCC là gì?

MVCC = Multi-Version Concurrency Control.

Thay vì chỉ có một phiên bản row, database có thể quản lý nhiều version để transaction đọc snapshot phù hợp.

Ý tưởng:

```text
Reader ----> version cũ phù hợp snapshot
Writer ----> tạo/update version mới
```

Điều này giúp reader và writer ít block nhau hơn so với locking thuần túy trong nhiều workload.

PostgreSQL sử dụng MVCC rất mạnh.

MVCC không có nghĩa là "không cần lock". Write conflict và một số operation vẫn cần locking/concurrency control.

---

# 5. Lock là gì?

Lock là cơ chế database dùng để kiểm soát truy cập đồng thời vào resource.

Hai loại khái niệm cơ bản:

- **Shared lock (S)**: thường phục vụ read; nhiều shared lock có thể cùng tồn tại tùy engine.
- **Exclusive lock (X)**: phục vụ write; conflict với các lock không tương thích.

Database còn có row lock, table lock, intention lock, predicate/range lock... tùy engine.

---

# 6. SELECT ... FOR UPDATE

Khi cần đọc một row rồi thay đổi nó dựa trên giá trị vừa đọc, có thể dùng pessimistic locking.

```sql
BEGIN;

SELECT stock
FROM products
WHERE id = 10
FOR UPDATE;

UPDATE products
SET stock = stock - 1
WHERE id = 10;

COMMIT;
```

`FOR UPDATE` yêu cầu lock phù hợp trên row được chọn để ngăn transaction khác thực hiện các thao tác xung đột cho đến khi transaction kết thúc, theo semantics của DB.

Use case:

- inventory
- wallet/balance
- job claiming
- resource allocation

---

# 7. Pessimistic vs Optimistic Locking

## Pessimistic Locking

Giả định conflict có khả năng xảy ra nên lock trước.

Ví dụ:

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

Ưu điểm:

- dễ reasoning khi contention cao
- ngăn một số race condition trực tiếp

Nhược điểm:

- blocking
- deadlock
- giảm concurrency nếu giữ lock lâu

## Optimistic Locking

Không lock resource trong toàn bộ khoảng xử lý; khi update sẽ kiểm tra xem dữ liệu có bị thay đổi hay chưa.

Ví dụ có column `version`:

```sql
UPDATE products
SET stock = 9,
    version = version + 1
WHERE id = 1
  AND version = 5;
```

Nếu affected rows = 0 thì version đã thay đổi -> conflict -> retry hoặc báo lỗi.

Phù hợp khi conflict tương đối hiếm.

---

# 8. Atomic Update

Nhiều bài toán không cần read-then-write.

Thay vì:

```text
SELECT stock
stock = stock - 1
UPDATE stock
```

có thể dùng một statement atomic:

```sql
UPDATE products
SET stock = stock - 1
WHERE id = 1
  AND stock > 0;
```

Sau đó kiểm tra affected rows.

Đây thường là giải pháp đơn giản và hiệu quả cho inventory counter.

---

# 9. Deadlock là gì?

Deadlock xảy ra khi các transaction chờ lock của nhau tạo thành vòng tròn.

Ví dụ:

```text
Transaction A:
  lock row 1
  chờ row 2

Transaction B:
  lock row 2
  chờ row 1
```

```text
A ----wait----> B
^               |
|               |
+-----wait-------+
```

Không transaction nào tự tiến tiếp được.

Database thường phát hiện deadlock và chọn một transaction làm victim để rollback/abort.

Application nên xử lý lỗi và retry transaction khi phù hợp.

---

# 10. Cách giảm Deadlock

## 1. Lock resource theo cùng thứ tự

Không nên:

```text
A: lock user 1 -> user 2
B: lock user 2 -> user 1
```

Nên thống nhất:

```text
A: lock user 1 -> user 2
B: lock user 1 -> user 2
```

## 2. Transaction ngắn

Giữ lock càng lâu thì contention/deadlock càng dễ xảy ra.

## 3. Có index phù hợp

Query update/delete không có index phù hợp có thể scan/lock nhiều dữ liệu hơn cần thiết tùy database.

## 4. Retry

Deadlock không phải lúc nào cũng có thể loại bỏ hoàn toàn.

Backend production nên có retry policy hợp lý với backoff và giới hạn số lần retry.

---

# 11. Lock khác Transaction thế nào?

```text
Transaction = ranh giới logic của một đơn vị công việc
Lock        = một cơ chế concurrency control
```

Transaction có thể khiến database acquire/release lock, nhưng hai khái niệm không giống nhau.

---

# 12. Isolation cao nhất có phải luôn tốt nhất?

Không.

`SERIALIZABLE` cung cấp guarantees mạnh nhưng có thể tăng blocking hoặc serialization failures/retry tùy implementation.

Chọn isolation level dựa trên:

- correctness requirement
- contention
- workload read/write
- DB engine
- khả năng retry của application

---

# Câu hỏi phỏng vấn

## Q1. Dirty read là gì?

Đọc dữ liệu của transaction khác khi dữ liệu đó chưa commit.

## Q2. Non-repeatable read là gì?

Cùng một row được đọc hai lần trong một transaction nhưng giá trị khác nhau do transaction khác commit update ở giữa.

## Q3. Phantom read là gì?

Cùng một query predicate chạy lại nhưng tập row thay đổi do transaction khác insert/delete dữ liệu phù hợp.

## Q4. Isolation level nào mạnh nhất?

`SERIALIZABLE` trong SQL standard.

## Q5. MVCC là gì?

Multi-Version Concurrency Control: database quản lý nhiều version của row để transaction có thể đọc snapshot phù hợp và giảm read/write blocking trong nhiều trường hợp.

## Q6. SELECT FOR UPDATE dùng khi nào?

Khi cần pessimistically lock row được đọc trước khi thực hiện logic/update phụ thuộc vào giá trị đó.

## Q7. Optimistic locking hoạt động thế nào?

Thường dùng version/timestamp trong điều kiện update. Nếu version không còn đúng thì update thất bại và application xử lý conflict/retry.

## Q8. Deadlock là gì?

Hai hoặc nhiều transaction giữ resource và chờ resource của nhau theo vòng tròn.

## Q9. Làm sao giảm deadlock?

Giữ transaction ngắn, lock resource theo thứ tự nhất quán, truy vấn/index hợp lý và có retry strategy.

## Q10. Lost update xử lý thế nào?

Tùy use case: atomic update, pessimistic lock (`FOR UPDATE`), optimistic locking hoặc isolation/concurrency mechanism phù hợp.

## Q11. Isolation cao hơn có luôn nhanh hơn không?

Không. Guarantee mạnh hơn thường có trade-off về concurrency, blocking hoặc retry.

## Q12. Deadlock có phải bug của database không?

Không. Nó là tình huống có thể xuất hiện trong concurrent transactions. Database thường detect và abort một transaction; application cần thiết kế lock order và retry phù hợp.

---

# Cheat Sheet

```text
Anomalies:
Dirty read          = đọc uncommitted data
Non-repeatable read = cùng row, đọc lại ra giá trị khác
Phantom read        = cùng predicate, tập row thay đổi
Lost update         = update bị update khác ghi đè

Isolation:
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE

Concurrency tools:
MVCC
Pessimistic lock -> SELECT ... FOR UPDATE
Optimistic lock  -> version column
Atomic update    -> UPDATE ... SET x = x - 1

Deadlock:
A giữ X, chờ Y
B giữ Y, chờ X

Giảm deadlock:
- consistent lock order
- short transaction
- good indexes/queries
- retry with backoff
```
