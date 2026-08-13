# SQL JOIN — Backend Interview Notes

## JOIN là gì?

`JOIN` kết hợp dữ liệu từ nhiều bảng dựa trên điều kiện liên quan.

```sql
SELECT u.id, u.name, o.id AS order_id
FROM users u
JOIN orders o ON o.user_id = u.id;
```

## Các loại JOIN

### INNER JOIN
Chỉ lấy các dòng match ở cả hai bảng.

```sql
SELECT u.name, o.id
FROM users u
INNER JOIN orders o ON o.user_id = u.id;
```

> INNER = phần giao nhau.

### LEFT JOIN
Giữ toàn bộ bảng trái. Nếu bảng phải không match, các cột phía phải là `NULL`.

```sql
SELECT u.name, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

> LEFT = giữ tất cả bên trái.

### RIGHT JOIN
Giữ toàn bộ bảng phải. Trong thực tế thường có thể đổi thứ tự bảng và dùng LEFT JOIN để dễ đọc hơn.

### FULL OUTER JOIN
Giữ dữ liệu từ cả hai phía; phần không match nhận `NULL` ở phía còn lại. Không phải DBMS nào cũng hỗ trợ trực tiếp.

### CROSS JOIN
Cartesian Product: mỗi row A kết hợp với mọi row B.

```text
100 rows × 200 rows = 20,000 rows
```

Cần cẩn thận vì kết quả có thể tăng cực nhanh.

### SELF JOIN
Một bảng JOIN với chính nó. Ví dụ employee và manager:

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;
```

## JOIN và Index

Với bảng lớn, các cột JOIN/filter thường cần index phù hợp.

```sql
SELECT *
FROM orders o
JOIN users u ON u.id = o.user_id;
```

`users.id` thường đã có index nếu là Primary Key. `orders.user_id` thường nên có index nếu được JOIN/filter thường xuyên.

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

Không nên thêm index máy móc: hãy kiểm tra execution plan và workload thực tế.

## JOIN Algorithms

### Nested Loop Join

```text
for each row in A:
    find matching rows in B
```

Có thể hiệu quả khi một phía ít row và phía còn lại có index tốt.

### Hash Join
DB xây hash table cho một input rồi lookup input còn lại. Thường phù hợp với equality JOIN trên tập dữ liệu lớn.

### Merge Join
Hai input được xử lý theo thứ tự join key rồi merge. Có thể hiệu quả khi dữ liệu đã có thứ tự phù hợp.

Optimizer thường tự chọn algorithm dựa trên statistics và estimated cost.

## Bẫy LEFT JOIN + WHERE

```sql
SELECT *
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'paid';
```

`WHERE o.status = 'paid'` loại các row mà `o` là NULL, nên kết quả có thể gần giống INNER JOIN.

Nếu muốn giữ tất cả users:

```sql
SELECT *
FROM users u
LEFT JOIN orders o
  ON o.user_id = u.id
 AND o.status = 'paid';
```

## N+1 Query Problem

Application lấy 100 users bằng 1 query rồi chạy thêm 1 query orders cho từng user:

```text
1 + 100 = 101 queries
```

Giải pháp tùy tình huống: JOIN, eager loading, batch query hoặc DataLoader pattern.

# Interview Questions

### INNER JOIN và LEFT JOIN khác nhau?
INNER chỉ trả row match hai phía. LEFT giữ toàn bộ row phía trái.

### JOIN chậm thì kiểm tra gì?
1. `EXPLAIN` / execution plan.
2. Index trên join/filter columns.
3. Cardinality và statistics.
4. Số row thực tế.
5. JOIN algorithm.
6. Có lấy quá nhiều dữ liệu hay không.

### Có nên index Foreign Key?
Không phải DBMS nào cũng tự tạo index cho FK. Nếu FK thường xuyên JOIN/filter hoặc tham gia thao tác parent-child, index thường hữu ích.

### LEFT JOIN + WHERE có thể thành INNER JOIN về mặt kết quả không?
Có. Điều kiện WHERE trên cột phía phải có thể loại các NULL row do LEFT JOIN tạo ra.

### CROSS JOIN nguy hiểm ở đâu?
Số row tăng theo tích số row hai bảng.

### N+1 là gì?
Một query lấy danh sách rồi phát sinh thêm N query cho N phần tử, gây nhiều database round-trip.

# Cheat Sheet

```text
INNER -> chỉ match
LEFT  -> giữ bên trái
RIGHT -> giữ bên phải
FULL  -> giữ cả hai
CROSS -> Cartesian product
SELF  -> JOIN chính table đó

JOIN chậm:
EXPLAIN -> index -> statistics/cardinality -> rows -> algorithm

LEFT JOIN + WHERE right.column = ...
-> coi chừng loại NULL

N+1
-> 1 query + N query con
```

## Câu trả lời phỏng vấn ngắn

> JOIN dùng để kết hợp dữ liệu từ nhiều bảng dựa trên điều kiện liên quan. Khi JOIN chậm, em kiểm tra execution plan, index trên join/filter columns, cardinality/statistics và lượng dữ liệu thực tế thay vì chỉ thêm index một cách máy móc.
