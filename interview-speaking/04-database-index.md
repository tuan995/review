# 04 — Database & Index

Mục tiêu của chương này là hiểu **index giúp gì, khi nào không giúp và cách nói từ query thực tế**, thay vì chỉ nhớ “index làm query nhanh hơn”.

---

# 1. Index là gì?

## 💬 Bài nói

> Em hiểu **index** là một cấu trúc dữ liệu phụ giúp database tìm dữ liệu nhanh hơn thay vì phải đọc toàn bộ table trong nhiều trường hợp.
>
> Với relational database, B-Tree là loại index rất phổ biến. Ví dụ nếu bảng `users` có một triệu row và thường tìm theo `email`, index trên `email` có thể giúp database tìm nhanh hơn so với scan toàn bộ bảng.
>
> Tuy nhiên index không phải càng nhiều càng tốt. Mỗi index tốn storage và khi insert/update/delete thì database cũng phải cập nhật index. Vì vậy em thường bắt đầu từ query thực tế, sau đó dùng `EXPLAIN` hoặc `EXPLAIN ANALYZE` để xem database đang chạy query như thế nào.

---

# 🧾 Thuật ngữ

### **Index** *(cấu trúc dữ liệu phụ giúp database tìm row hiệu quả hơn)*

### **Table scan / Sequential scan** *(database đọc nhiều hoặc toàn bộ row của table để tìm kết quả)*

### **B-Tree** *(cấu trúc cây cân bằng thường được database dùng cho index)*

Khi phỏng vấn không cần mô tả thuật toán B-Tree quá sâu nếu chưa được hỏi. Có thể nói:

> B-Tree giữ dữ liệu index theo cấu trúc có thứ tự, nhờ vậy việc tìm kiếm/range lookup hiệu quả hơn việc scan toàn bộ table.

### **Execution plan** *(kế hoạch database chọn để thực thi query)*

Nó cho biết database định scan table, dùng index, join theo cách nào...

### **Query optimizer** *(bộ phận database chọn execution plan dựa trên cost ước lượng)*

---

# 2. Có index là database chắc chắn dùng không?

Không.

## 📌 Ví dụ

Bảng có 1.000.000 user và index trên `department_id`.

- Department A có 100 user.
- Department B có 800.000 user.

Với A, dùng index có thể rất hiệu quả vì chỉ lấy một phần rất nhỏ table.

Với B, database có thể thấy đọc phần lớn table qua index còn tốn hơn scan trực tiếp, nên optimizer có thể chọn sequential scan.

## **Selectivity** *(mức độ một điều kiện lọc thu hẹp dữ liệu)*

Điều kiện càng chọn ra ít row so với toàn table thì thường càng có selectivity tốt.

⚠️ **Dễ bị bắt bẻ:**

> “Column selectivity cao thì database luôn dùng index.”

✅ **Cách nói an toàn:**

> Selectivity là một yếu tố quan trọng nhưng optimizer còn xét statistics, cost, số row, cách query và loại index.

---

# 3. `EXPLAIN` và `EXPLAIN ANALYZE`

## 💬 Bài nói

> Khi query chậm em không thêm index theo cảm tính. Em xem execution plan trước để biết database đang thực sự làm gì.
>
> `EXPLAIN` cho em kế hoạch mà database dự định dùng. `EXPLAIN ANALYZE` còn thực thi query và cho thông tin runtime thực tế, nên em dùng cẩn thận với query nặng hoặc query có side effect tùy database.

### **Estimated cost** *(chi phí database ước lượng cho một plan)*

Không phải thời gian milliseconds trực tiếp. Nó là giá trị tương đối để optimizer so sánh các plan.

### **Statistics** *(thông tin database lưu về phân bố dữ liệu để optimizer ước lượng)*

Nếu statistics không phản ánh dữ liệu thực tế, optimizer có thể estimate sai.

---

# 4. Composite Index

## **Composite index** *(index gồm nhiều column)*

Ví dụ:

```sql
CREATE INDEX idx_users_department_role
ON users(department_id, role_id);
```

## 💬 Bài nói

> Với composite index, thứ tự column quan trọng. Ví dụ index `(department_id, role_id)` thường hỗ trợ tốt query bắt đầu từ `department_id`, hoặc `department_id` kết hợp `role_id`.
>
> Query chỉ lọc `role_id` thường không tận dụng index này theo cách B-Tree thông thường vì column đầu tiên không có trong điều kiện.

### **Leftmost prefix** *(quy tắc dùng các column bắt đầu từ phía trái của composite index)*

Với `(A, B, C)`:

- A → có thể dùng.
- A + B → có thể dùng tốt.
- A + B + C → có thể dùng.
- chỉ B → thường không dùng index đó theo cách thông thường.

⚠️ Đây là mental model. Một số database có optimization khác, nên tránh nói “không bao giờ”.

---

# 5. Chọn thứ tự column trong composite index thế nào?

## 💬 Cách nói

> Em không chọn chỉ dựa trên column nào unique hơn. Em nhìn query pattern thực tế: column nào thường equality, column nào range, có sort không, và query nào quan trọng nhất. Sau đó em validate bằng execution plan.

## 📌 Ví dụ

Query thường là:

```sql
SELECT *
FROM orders
WHERE shop = ?
  AND created_at >= ?
ORDER BY created_at DESC;
```

Một index `(shop, created_at)` có thể hợp lý vì query thường lọc một `shop` cụ thể rồi range/sort theo `created_at`.

---

# 6. LIKE và Index

## Prefix search

```sql
WHERE name LIKE 'abc%'
```

Với B-Tree và điều kiện phù hợp, database có thể tận dụng thứ tự prefix.

## Leading wildcard

```sql
WHERE name LIKE '%abc'
```

Database không biết bắt đầu từ phần nào của B-Tree nên index thông thường thường khó giúp hiệu quả.

### Nếu cần search phức tạp?

Có thể cân nhắc:

- full-text search;
- trigram index tùy database;
- Elasticsearch/OpenSearch cho bài toán search chuyên dụng.

⚠️ Không nên nói “`%abc` tuyệt đối không dùng index” cho mọi database/index type.

---

# 7. Function trên indexed column

Ví dụ:

```sql
WHERE LOWER(email) = 'a@example.com'
```

Nếu chỉ có index trên `email`, database có thể không dùng nó hiệu quả vì query đang tìm theo kết quả của `LOWER(email)`.

### **Expression / Functional Index** *(index trên kết quả của expression/function)*

Ví dụ tùy database có thể tạo index trên `LOWER(email)`.

✅ **Cách nói an toàn:**

> Function trên indexed column có thể làm index bình thường không phù hợp. Nếu query đó phổ biến, em xem database có hỗ trợ expression index hay không.

---

# 8. Clustered vs Non-clustered Index

Phần này dễ bị hỏi khác nhau theo database, nên nói cẩn thận.

### **Clustered index** *(dữ liệu table được tổ chức vật lý/logical theo index chính tùy engine)*

### **Non-clustered / Secondary index** *(index riêng chứa key và cách trỏ tới row dữ liệu)*

⚠️ Cách implementation khác nhau giữa SQL Server, MySQL/InnoDB, PostgreSQL...

✅ **Cách nói an toàn:**

> Khái niệm clustered phụ thuộc database engine. Khi interview em sẽ nói cụ thể theo engine đang được hỏi thay vì đưa một định nghĩa áp cho tất cả database.

---

# 9. Index có nhược điểm gì?

- Tốn disk/storage.
- Insert/update/delete phải maintain index.
- Quá nhiều index làm write nặng hơn.
- Một index có thể tốt cho query này nhưng không giúp query khác.
- Schema thay đổi và workload thay đổi có thể làm index cũ không còn hữu ích.

### **Write overhead** *(chi phí thêm khi ghi dữ liệu vì database phải cập nhật index)*

---

# 10. Case thực tế: Query chậm

## 💬 Bài nói 60–90 giây

> Khi gặp query chậm em không kết luận ngay là thiếu index. Đầu tiên em xác định query nào chậm, input nào chậm và số lượng dữ liệu thực tế.
>
> Sau đó em xem execution plan để biết database đang scan table hay dùng index, số row estimate có hợp lý không và join có tạo nhiều dữ liệu trung gian không.
>
> Nếu vấn đề là thiếu index thì em thiết kế index dựa trên filter, join và sort của query. Sau khi thêm em chạy lại plan và benchmark để kiểm tra improvement.
>
> Em cũng xem write overhead vì một index mới có thể giúp read nhưng làm insert/update chậm hơn.

---

# 🎯 Interviewer hỏi tiếp

### Query chậm có phải luôn do thiếu index?

> Không. Có thể do N+1 query, lấy quá nhiều data, lock contention, connection pool, join không phù hợp, network, schema hoặc application gọi query quá nhiều lần.

### N+1 là gì?

> Ví dụ em query 100 orders một lần, sau đó trong loop lại query line items cho từng order thành 100 query nữa. Tổng cộng 101 query. Thường có thể giảm bằng join/include/batch tùy ORM và use case.

### Covering index là gì?

> Là index chứa đủ những column query cần để database có thể trả kết quả từ index mà không phải quay lại đọc table/heap trong một số engine. Lợi ích và cách hoạt động phụ thuộc database.

### Unique index khác gì normal index?

> Unique index vừa hỗ trợ lookup vừa enforce không cho hai row có cùng key theo rule của database.

---

# ⚠️ Những câu dễ bị bắt bẻ

❌ “Index luôn làm query nhanh.”  
✅ “Index có thể giúp nhiều query đọc nhanh hơn, nhưng optimizer chỉ dùng khi cost phù hợp và index cũng làm write tốn thêm chi phí.”

❌ “Composite index phải để column selectivity cao nhất trước.”  
✅ “Em chọn thứ tự dựa vào query pattern thực tế, equality/range/sort và validate bằng execution plan.”

❌ “LIKE `%abc` không bao giờ dùng index.”  
✅ “B-Tree thông thường khó tận dụng leading wildcard; database có thể có index/search mechanism khác.”

---

# 📌 Cách nhớ

**Query thực tế → EXPLAIN → số row/selectivity → index phù hợp → benchmark → xem write cost**

Đây là flow an toàn hơn việc thấy query chậm rồi “thêm index” ngay.