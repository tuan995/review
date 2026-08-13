# SQL Query Optimization & EXPLAIN

## 1. Mục tiêu

Tối ưu query không phải là “thêm index thật nhiều”. Mục tiêu là giảm lượng dữ liệu phải đọc, giảm I/O, giảm CPU, giảm sort/hash không cần thiết và giúp optimizer chọn execution plan tốt.

## 2. EXPLAIN là gì?

`EXPLAIN` cho biết database dự định chạy query như thế nào.

```sql
EXPLAIN
SELECT *
FROM users
WHERE department_id = 10;
```

`EXPLAIN ANALYZE` thường chạy query thật và trả thêm số liệu thực tế như thời gian, số dòng thực tế, loops.

```sql
EXPLAIN ANALYZE
SELECT *
FROM users
WHERE department_id = 10;
```

> Khi tối ưu: đo trước -> thay đổi -> đo lại.

## 3. Những node thường gặp

Tên cụ thể tùy DBMS, nhưng thường có:

- Sequential/Table Scan: quét bảng.
- Index Scan/Seek: dùng index để tìm dữ liệu.
- Index Only Scan: có thể trả dữ liệu từ index mà không cần đọc bảng trong một số trường hợp.
- Nested Loop: phù hợp khi một phía nhỏ hoặc lookup theo index hiệu quả.
- Hash Join: thường hiệu quả khi join tập dữ liệu lớn theo equality.
- Merge Join: hữu ích khi hai phía đã có thứ tự phù hợp.
- Sort: cần sắp xếp dữ liệu.
- Aggregate/HashAggregate: xử lý GROUP BY/aggregate.

## 4. Vì sao có index mà vẫn chậm?

- Query trả về quá nhiều dòng.
- Index sai thứ tự cột.
- Selectivity thấp.
- Dùng hàm/cast khiến điều kiện không tận dụng index tốt.
- `LIKE '%abc%'` với B-tree thông thường.
- Join trên cột không có index phù hợp.
- Sort/Group By trên lượng dữ liệu lớn.
- Chọn quá nhiều cột bằng `SELECT *`.
- N+1 query ở tầng ứng dụng.
- Statistics cũ hoặc ước lượng cardinality sai.

## 5. SARGable condition

Một điều kiện “SARGable” là điều kiện database có thể dùng index hiệu quả.

Kém hiệu quả:

```sql
WHERE YEAR(created_at) = 2026
```

Thường tốt hơn:

```sql
WHERE created_at >= '2026-01-01'
  AND created_at <  '2027-01-01'
```

Ý tưởng: tránh biến đổi trực tiếp cột đã đánh index nếu có thể viết thành range.

## 6. Chỉ lấy dữ liệu cần thiết

Không nên mặc định:

```sql
SELECT * FROM users;
```

Nếu chỉ cần id và email:

```sql
SELECT id, email FROM users;
```

Lợi ích: ít I/O, ít network, có cơ hội dùng covering/index-only scan.

## 7. Pagination

Offset lớn có thể chậm:

```sql
SELECT *
FROM orders
ORDER BY id
LIMIT 20 OFFSET 1000000;
```

Keyset pagination thường tốt hơn:

```sql
SELECT *
FROM orders
WHERE id > :last_id
ORDER BY id
LIMIT 20;
```

## 8. N+1 query

Ví dụ ứng dụng lấy 100 users rồi chạy thêm 100 query lấy department.

Hướng xử lý:

- JOIN phù hợp.
- Batch query.
- DataLoader.
- Eager loading có kiểm soát.

## 9. Index cho JOIN

Ví dụ:

```sql
SELECT u.id, d.name
FROM users u
JOIN departments d ON d.id = u.department_id;
```

Thông thường `departments.id` là PK đã có index; `users.department_id` thường nên có index nếu join/filter theo cột này thường xuyên.

## 10. Composite index và ORDER BY

Query:

```sql
SELECT *
FROM users
WHERE department_id = ?
ORDER BY created_at DESC;
```

Có thể cân nhắc:

```sql
CREATE INDEX idx_users_department_created
ON users(department_id, created_at);
```

Thứ tự cột phải dựa trên query thực tế và xác minh bằng EXPLAIN.

## 11. Quy trình tối ưu query

1. Xác định query chậm bằng metric/APM/slow query log.
2. Chạy EXPLAIN / EXPLAIN ANALYZE.
3. Xem số dòng, scan, join, sort, filter.
4. Kiểm tra index hiện có.
5. Giảm dữ liệu đọc hoặc đổi query/index.
6. Test lại trên dữ liệu gần production.
7. Đo lại latency, I/O, CPU và throughput.

## 12. Câu hỏi phỏng vấn

### Làm sao biết query có dùng index?
Dùng EXPLAIN/EXPLAIN ANALYZE và xem execution plan.

### Vì sao optimizer chọn table scan dù có index?
Vì optimizer ước tính scan toàn bảng rẻ hơn, ví dụ query trả về phần lớn dữ liệu hoặc index có selectivity thấp.

### Vì sao `SELECT *` có thể chậm?
Vì đọc nhiều dữ liệu hơn, truyền nhiều dữ liệu hơn và có thể làm mất cơ hội index-only scan.

### Offset pagination có vấn đề gì?
Offset càng lớn, DB càng phải bỏ qua nhiều dòng. Keyset pagination thường scale tốt hơn.

### Tối ưu query bắt đầu từ đâu?
Từ đo lường và execution plan, không bắt đầu bằng việc tạo index ngẫu nhiên.

## Cheat sheet

- EXPLAIN trước khi đoán.
- Index dựa trên query thực tế.
- Tránh đọc dữ liệu không cần thiết.
- Tránh function/cast trên indexed column khi có thể viết thành range.
- Chú ý selectivity và cardinality.
- Tránh N+1.
- Offset lớn -> cân nhắc keyset pagination.
- Tối ưu xong phải đo lại.
