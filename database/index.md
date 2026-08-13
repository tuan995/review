# Database Index — Backend Interview Notes

## 1. Index là gì?

Index là một cấu trúc dữ liệu trong database giúp database tìm dữ liệu nhanh hơn thay vì phải quét toàn bộ bảng (full table scan).

Loại index phổ biến nhất là **B-tree / B+Tree**.

Có thể hình dung index giống mục lục của một cuốn sách: thay vì đọc từ trang đầu đến trang cuối, ta dùng mục lục để đi gần như trực tiếp đến vị trí cần tìm.

### Trade-off

Index thường giúp **đọc nhanh hơn**, nhưng đổi lại:

- Tốn thêm dung lượng lưu trữ.
- `INSERT` chậm hơn vì phải thêm dữ liệu vào index.
- `UPDATE` có thể chậm hơn nếu cập nhật cột được index.
- `DELETE` phải xóa/cập nhật index tương ứng.
- Quá nhiều index làm tăng chi phí bảo trì.

> Mẹo nhớ: **Index = đọc nhanh hơn, ghi đắt hơn.**

---

## 2. Khi nào nên tạo Index?

Không nên tạo index chỉ vì một cột "có vẻ quan trọng". Hãy thiết kế index dựa trên **mẫu truy vấn thực tế**.

Thường cân nhắc index cho cột xuất hiện nhiều trong:

- `WHERE`
- `JOIN`
- `ORDER BY`
- đôi khi `GROUP BY`
- các ràng buộc `UNIQUE`, ví dụ email đăng nhập

Ví dụ:

```sql
SELECT *
FROM users
WHERE email = 'user@example.com';
```

Nếu `email` thường xuyên được dùng để đăng nhập:

```sql
CREATE UNIQUE INDEX idx_users_email
ON users(email);
```

Sau khi tạo index, dùng `EXPLAIN` hoặc `EXPLAIN ANALYZE` để kiểm tra execution plan thay vì đoán.

---

## 3. Tại sao có Index mà Database vẫn không dùng?

Có index **không có nghĩa** optimizer bắt buộc phải dùng nó.

Optimizer ước tính chi phí của nhiều execution plan và chọn plan được cho là rẻ nhất.

### Trường hợp 1: Query trả về quá nhiều dòng

Giả sử bảng `users` có 1.000.000 bản ghi và có index:

```sql
CREATE INDEX idx_users_department
ON users(department_id);
```

Nếu:

```sql
SELECT * FROM users WHERE department_id = 10;
```

chỉ trả về khoảng 100 dòng, index có thể rất hiệu quả.

Nhưng nếu một `department_id` khớp 800.000/1.000.000 dòng, database có thể chọn full table scan.

Lý do: đi qua index rồi đọc một lượng cực lớn row có thể gây nhiều random I/O; đọc tuần tự toàn bảng đôi khi rẻ hơn.

### Trường hợp 2: Dùng hàm trên cột

Ví dụ:

```sql
SELECT *
FROM users
WHERE YEAR(created_at) = 2026;
```

Với index thông thường trên `created_at`, việc bọc cột trong hàm có thể khiến database không tận dụng index theo cách mong muốn (tùy DB và loại index).

Một cách viết thường thân thiện với B-tree hơn:

```sql
SELECT *
FROM users
WHERE created_at >= '2026-01-01'
  AND created_at <  '2027-01-01';
```

### Trường hợp 3: Composite index không khớp

Nếu có:

```sql
CREATE INDEX idx_users_department_role
ON users(department_id, role_id);
```

nhưng query chỉ có:

```sql
SELECT *
FROM users
WHERE role_id = 2;
```

thì index trên thường không hiệu quả cho query này vì thiếu cột dẫn đầu `department_id` (chi tiết ở phần Leftmost Prefix).

> Khi phỏng vấn: **Có index chưa đủ. Hãy kiểm tra execution plan bằng EXPLAIN.**

---

## 4. Composite Index là gì?

Composite Index là index gồm **nhiều cột**.

Ví dụ:

```sql
CREATE INDEX idx_users_department_role
ON users(department_id, role_id);
```

Phù hợp khi ứng dụng thường query:

```sql
SELECT *
FROM users
WHERE department_id = 10
  AND role_id = 2;
```

### Composite Index hay nhiều Index đơn?

Giả sử có hai index đơn:

```sql
CREATE INDEX idx_department ON users(department_id);
CREATE INDEX idx_role ON users(role_id);
```

và một lựa chọn khác:

```sql
CREATE INDEX idx_department_role
ON users(department_id, role_id);
```

Nếu phần lớn query dùng `department_id` và `role_id` cùng nhau, composite index thường là lựa chọn đáng cân nhắc.

Nếu ứng dụng thường xuyên query hai cột hoàn toàn độc lập, index đơn có thể linh hoạt hơn.

Không có đáp án cố định: **dựa vào workload thật + EXPLAIN + benchmark.**

---

## 5. Leftmost Prefix

Đây là quy tắc cực kỳ quan trọng với composite B-tree index.

Giả sử index:

```text
(department_id, role_id, created_at)
```

Các query bắt đầu từ phần bên trái thường tận dụng index tốt:

```text
department_id

department_id + role_id

department_id + role_id + created_at
```

Nhưng chỉ có:

```text
role_id
```

hoặc:

```text
created_at
```

thì index này thường không phục vụ tốt việc tìm kiếm đó.

### Ví dụ dễ nhớ

Index:

```text
(first_name, last_name)
```

Query:

```sql
WHERE first_name = 'John'
```

→ có thể tận dụng index.

```sql
WHERE first_name = 'John'
  AND last_name = 'Smith'
```

→ có thể tận dụng index rất tốt.

```sql
WHERE last_name = 'Smith'
```

→ composite index trên thường không hiệu quả cho việc lookup chỉ bằng `last_name`.

> Mẹo nhớ: **Composite index giống số điện thoại theo mã vùng: muốn đi nhanh, bắt đầu từ bên trái.**

Lưu ý: optimizer và từng hệ quản trị có thể có các kỹ thuật đặc biệt (ví dụ skip scan trong một số DB/trường hợp), vì vậy đây là nguyên tắc thiết kế chứ không nên biến thành khẳng định tuyệt đối rằng DB "không bao giờ" dùng index.

---

## 6. LIKE và Index

### `LIKE 'abc%'`

```sql
SELECT *
FROM users
WHERE username LIKE 'abc%';
```

Với B-tree index phù hợp, database thường có thể dùng phần prefix `abc` để xác định một khoảng giá trị cần tìm.

Do đó `LIKE 'abc%'` **có khả năng tận dụng index**.

### `LIKE '%abc'` hoặc `LIKE '%abc%'`

```sql
SELECT *
FROM users
WHERE username LIKE '%abc%';
```

Wildcard nằm ở đầu làm mất prefix cố định. B-tree thông thường không biết nên bắt đầu lookup ở vị trí nào, vì vậy thường phải kiểm tra rất nhiều giá trị.

### Hướng tối ưu

Nếu đây là tìm kiếm văn bản, cân nhắc:

- Full-text index / full-text search.
- PostgreSQL trigram index cho các use case substring phù hợp.
- Search engine chuyên dụng như Elasticsearch/OpenSearch khi yêu cầu search phức tạp và quy mô đủ lớn.

Không nên mặc định đưa Elasticsearch vào chỉ để xử lý một query nhỏ; luôn cân nhắc độ phức tạp vận hành.

---

## 7. Selectivity

Selectivity mô tả khả năng một điều kiện/index thu hẹp số lượng row.

Ví dụ bảng có 10 triệu user nhưng cột:

```text
gender = male | female
```

chỉ có vài giá trị khác nhau.

Nếu `male` chiếm khoảng 50% bảng, query:

```sql
WHERE gender = 'male'
```

vẫn phải đọc một lượng dữ liệu rất lớn. Index đơn trên `gender` có thể mang lại ít lợi ích cho một số query.

Ngược lại, `email` thường gần như duy nhất cho từng user nên có selectivity cao.

### Mẹo nhớ

- Giá trị càng phân biệt tốt → selectivity thường càng cao.
- Điều kiện trả về càng ít row → index lookup thường càng có cơ hội hữu ích.

Nhưng optimizer còn xem nhiều yếu tố khác, không chỉ selectivity.

---

## 8. Có phải càng nhiều Index càng tốt?

**Không.**

Mỗi index mới có thể:

- tăng storage;
- tăng chi phí `INSERT`;
- tăng chi phí `UPDATE`;
- tăng chi phí `DELETE`;
- cần maintenance;
- làm schema/index strategy phức tạp hơn.

Chỉ nên tạo index khi lợi ích đọc đáng với chi phí duy trì.

---

## 9. Clustered vs Non-clustered Index

Khái niệm này phụ thuộc khá nhiều vào hệ quản trị database, nên tránh học thuộc một định nghĩa áp dụng cho mọi DB.

### Cách hiểu phỏng vấn cơ bản

**Clustered index** liên quan chặt tới cách dữ liệu row được tổ chức/lưu trữ theo key của index. Vì dữ liệu chỉ có thể có một cách tổ chức chính, thường chỉ có một clustered organization/index cho một bảng.

**Non-clustered index** là cấu trúc index tách khỏi phần lưu row chính và chứa thông tin để tìm tới row tương ứng. Một bảng có thể có nhiều non-clustered/secondary indexes.

### Hình dung

Clustered:

```text
Dữ liệu thật được tổ chức theo key
1 -> row...
2 -> row...
3 -> row...
```

Non-clustered:

```text
Index riêng
key A -> vị trí/row identifier
key B -> vị trí/row identifier
```

> Lưu ý phỏng vấn: SQL Server, InnoDB/MySQL và PostgreSQL có cách tổ chức storage/index khác nhau. Nếu interviewer hỏi sâu, hãy xác định họ đang nói về DB nào.

---

## 10. B-tree là gì?

B-tree/B+Tree là cấu trúc cây cân bằng được thiết kế để tìm kiếm, chèn và xóa hiệu quả, đồng thời phù hợp với storage theo page/block của database.

Thay vì quét N bản ghi, lookup qua cây thường có độ phức tạp theo chiều cao cây, gần `O(log N)` về mặt mô hình thuật toán.

B-tree đặc biệt hữu ích cho:

- equality lookup (`=`)
- range query (`>`, `<`, `BETWEEN`)
- prefix/range phù hợp
- hỗ trợ ordering trong nhiều trường hợp

---

## 11. EXPLAIN / EXPLAIN ANALYZE

Muốn biết database có thực sự dùng index không, xem execution plan.

Ví dụ PostgreSQL:

```sql
EXPLAIN
SELECT * FROM users WHERE email = 'a@example.com';
```

Hoặc:

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'a@example.com';
```

Tùy database, bạn có thể gặp các operation như:

- Index Scan
- Index Only Scan
- Bitmap Index/Heap Scan
- Sequential Scan / Table Scan

Đừng tối ưu dựa hoàn toàn vào cảm giác.

Quy trình tốt:

```text
Query thực tế
    ↓
Đo performance
    ↓
EXPLAIN / execution plan
    ↓
Thiết kế/chỉnh index
    ↓
Đo lại
```

---

# Câu hỏi phỏng vấn

## Câu 1: Index là gì?

**Trả lời ngắn:**

> Index là cấu trúc dữ liệu giúp database tìm dữ liệu nhanh hơn và giảm nhu cầu quét toàn bảng. B-tree/B+Tree là loại phổ biến. Đổi lại index tốn storage và làm thao tác ghi đắt hơn.

---

## Câu 2: Tại sao có Index mà Database không dùng?

**Trả lời ngắn:**

> Optimizer chọn execution plan dựa trên estimated cost. Nếu query trả về quá nhiều row, index không phù hợp, dùng function/cast làm mất khả năng lookup, hoặc full scan được ước tính rẻ hơn thì optimizer có thể không chọn index. Tôi sẽ dùng EXPLAIN/EXPLAIN ANALYZE để kiểm chứng.

---

## Câu 3: Khi nào dùng Composite Index thay vì nhiều Index đơn?

> Khi workload thường xuyên lọc/sắp xếp theo cùng một nhóm cột, composite index thường đáng cân nhắc. Nếu các cột chủ yếu được query độc lập thì index đơn có thể phù hợp hơn. Quyết định cuối cùng dựa vào query pattern và execution plan.

---

## Câu 4: Leftmost Prefix là gì?

> Với composite B-tree index `(A, B, C)`, index thường phục vụ tốt các lookup bắt đầu từ phần bên trái như `A`, `(A,B)`, `(A,B,C)`. Chỉ query `B` hoặc `C` thường không tận dụng tốt index này.

---

## Câu 5: Vì sao `LIKE 'abc%'` và `LIKE '%abc%'` khác nhau?

> `abc%` có prefix cố định nên B-tree có thể xác định range cần tìm. `%abc%` không có prefix cố định nên B-tree thông thường không thể thực hiện lookup tương tự và thường phải kiểm tra nhiều dữ liệu hơn.

---

## Câu 6: Index có nhược điểm gì?

> Tốn storage và làm `INSERT`, `UPDATE`, `DELETE` đắt hơn vì database phải duy trì thêm index.

---

## Câu 7: Có phải càng nhiều Index càng tốt?

> Không. Index là trade-off giữa read performance và write/storage/maintenance cost. Chỉ tạo index phục vụ workload thực tế.

---

## Câu 8: Selectivity là gì?

> Selectivity thể hiện mức độ một điều kiện giúp thu hẹp tập dữ liệu. Cột/điều kiện có selectivity tốt thường phù hợp hơn với index lookup so với điều kiện trả về phần lớn bảng.

---

## Câu 9: Clustered và Non-clustered Index khác gì?

> Clustered index liên quan tới cách row được tổ chức theo key chính của index; non-clustered/secondary index là cấu trúc riêng giúp tìm tới row. Chi tiết implementation phụ thuộc DB engine.

---

## Câu 10: Làm sao biết query có dùng Index?

> Dùng `EXPLAIN` hoặc `EXPLAIN ANALYZE` và đọc execution plan.

---

# Cheat Sheet

```text
INDEX
│
├── Mục đích
│   └── Read nhanh hơn / giảm full scan
│
├── Trade-off
│   ├── Storage ↑
│   ├── INSERT cost ↑
│   ├── UPDATE cost ↑
│   └── DELETE cost ↑
│
├── Khi cân nhắc
│   ├── WHERE
│   ├── JOIN
│   ├── ORDER BY
│   ├── GROUP BY (tùy query)
│   └── UNIQUE
│
├── Composite Index
│   └── Nhớ Leftmost Prefix
│
├── LIKE
│   ├── 'abc%'  → thường có thể dùng B-tree
│   └── '%abc%' → B-tree thường không lookup hiệu quả
│
├── Selectivity
│   ├── Cao → thường thuận lợi hơn
│   └── Thấp → có thể full scan rẻ hơn
│
└── Kiểm chứng
    └── EXPLAIN / EXPLAIN ANALYZE
```

## Một câu chốt để đi phỏng vấn

> Tôi không tạo index chỉ dựa trên schema. Tôi nhìn vào query pattern và workload thực tế, thiết kế index cho các truy vấn quan trọng, sau đó dùng EXPLAIN/EXPLAIN ANALYZE và benchmark để xác nhận lợi ích, vì index luôn là trade-off giữa tốc độ đọc với chi phí ghi, storage và maintenance.
