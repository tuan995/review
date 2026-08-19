# 04 — Database & Index

# 1. Index là gì?

## Bài nói

> Em hiểu index giống một cấu trúc dữ liệu phụ giúp database tìm row nhanh hơn thay vì phải scan toàn bộ table. Với relational database, B-Tree là loại index rất phổ biến.
>
> Tuy nhiên em không coi index là thứ cứ thêm càng nhiều càng tốt. Mỗi index tốn storage và database phải maintain nó khi insert/update/delete. Vì vậy em thường bắt đầu từ query thực tế: `WHERE`, `JOIN`, `ORDER BY`, số lượng row và selectivity, sau đó dùng `EXPLAIN` hoặc `EXPLAIN ANALYZE` để kiểm tra execution plan.

### Tại sao database không dùng index dù có index?

> Optimizer chọn plan dựa trên cost. Nếu query trả về phần lớn table thì sequential scan có thể rẻ hơn việc đi qua index rồi đọc rất nhiều rows.

---

# 2. Selectivity

## Bài nói

> Ví dụ bảng có một triệu user và index `department_id`. Nếu một department chỉ có khoảng 100 user thì index có selectivity tốt cho query đó. Nhưng nếu một giá trị chiếm 800 nghìn rows thì index có thể không mang lại lợi ích và optimizer có thể chọn full scan.

### Có index là chắc chắn nhanh?

> Không. Phụ thuộc data distribution, query pattern, statistics và cost estimation.

---

# 3. Composite Index

## Bài nói

> Composite index là index trên nhiều column, ví dụ `(department_id, role_id)`. Thứ tự column quan trọng vì B-Tree composite index thường hiệu quả theo leftmost prefix.
>
> Với `(department_id, role_id)`, query theo `department_id` có thể tận dụng index. Query chỉ theo `role_id` thường không tận dụng index này theo cách thông thường.

### Chọn thứ tự column thế nào?

> Không có rule chỉ dựa vào “column nào unique hơn”. Em nhìn query pattern thực tế: equality/range conditions, sort và cách application truy vấn. Sau đó validate bằng execution plan.

---

# 4. LIKE và function

> Với B-Tree thông thường, prefix search như `LIKE 'abc%'` có khả năng tận dụng index trong điều kiện phù hợp, còn `LIKE '%abc'` không thể seek theo prefix theo cách đó. Với full-text/search phức tạp em cân nhắc full-text index, trigram hoặc search engine.

> Tương tự, nếu bọc indexed column bằng function, ví dụ `LOWER(name)`, index bình thường trên `name` có thể không phù hợp. Tùy database có thể dùng functional/expression index.

---

# 5. Case thực tế: query chậm

## Bài nói

> Khi gặp query chậm em không thêm index ngay. Đầu tiên em xác định query thực tế, số row và thời gian. Sau đó xem execution plan để biết đang sequential scan, index scan hay join theo cách nào.
>
> Nếu bottleneck là thiếu index, em thiết kế index dựa trên filter/join/order pattern. Sau khi thêm em chạy lại plan và benchmark. Đồng thời em cân nhắc write overhead vì index mới sẽ ảnh hưởng insert/update.

### Query chậm có phải luôn do thiếu index?

> Không. Có thể do N+1, query lấy quá nhiều data, lock contention, bad join, network, connection pool hoặc schema/query design.

## Cách nhớ

`Query → EXPLAIN → selectivity → index → benchmark → write trade-off`
