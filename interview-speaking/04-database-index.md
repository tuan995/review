# 04 — Database & Index

> Mục tiêu: nói được cách mình **phát hiện query chậm và quyết định có cần index hay không**, không chỉ đọc định nghĩa B-Tree.

# 1. Index là gì?

## Bài nói

> Em hiểu index giống như một cấu trúc phụ giúp database tìm dữ liệu nhanh hơn mà không phải đọc toàn bộ table trong nhiều trường hợp.
>
> Ví dụ bảng `users` có một triệu row. Nếu thường xuyên tìm theo `email`, một index phù hợp có thể giúp database đi tới nhóm dữ liệu cần tìm nhanh hơn thay vì kiểm tra từng row.
>
> Nhưng em không thêm index càng nhiều càng tốt. Mỗi index tốn storage và mỗi lần insert/update/delete database cũng phải cập nhật index. Vì vậy em thường nhìn query thực tế trước rồi mới quyết định.

### B-Tree là gì?

> B-Tree là cấu trúc dữ liệu dạng cây được nhiều relational database dùng cho index. Khi phỏng vấn em không cố giải thích thuật toán cây quá sâu nếu câu hỏi chỉ là cách dùng index. Ý chính là dữ liệu trong index được tổ chức để database tìm theo giá trị/range hiệu quả hơn việc scan toàn bộ table.

### “Scan toàn bộ table” là gì?

> Nghĩa là database phải đọc rất nhiều hoặc toàn bộ row để tìm row phù hợp. Trong execution plan thường có thể thấy dạng sequential scan/full table scan tùy database.

---

# 2. Làm sao biết query có dùng index?

## Bài nói

> Khi query chậm, em không đoán ngay là thiếu index. Em dùng `EXPLAIN` hoặc `EXPLAIN ANALYZE` để xem database dự định hoặc thực tế thực hiện query như thế nào.
>
> Em quan tâm database đang đọc toàn bộ table hay dùng index, số row ước lượng/thực tế, join nào tốn nhiều và thời gian nằm ở đâu. Sau khi thay đổi query hoặc index em chạy lại để so sánh.

### Execution plan là gì?

> Là kế hoạch database dùng để thực hiện query, ví dụ đọc table bằng cách nào, dùng index nào, join các bảng theo thứ tự nào. Có thể hiểu nó như “database định làm query này theo các bước nào”.

### Optimizer là gì?

> Là phần của database chọn execution plan mà nó ước tính là có chi phí thấp hơn. Vì vậy có index không có nghĩa database bắt buộc phải dùng index đó.

### Tại sao có index mà DB vẫn không dùng?

> Ví dụ query cần lấy 80% table. Đi qua index rồi vẫn phải đọc gần như toàn bộ dữ liệu, nên database có thể thấy đọc tuần tự toàn table rẻ hơn.

---

# 3. Selectivity — nói đơn giản thế nào?

> Nếu dùng từ selectivity, em giải thích là **điều kiện lọc có thu hẹp dữ liệu được nhiều hay không**.
>
> Ví dụ có một triệu user. `department_id = 10` chỉ trả 100 user thì điều kiện lọc khá hẹp, index thường có ích. Nhưng nếu `status = 'ACTIVE'` trả 900 nghìn user thì index trên `status` có thể không giúp nhiều cho query đó.

### Có index là chắc chắn nhanh?

> Không. Còn phụ thuộc query trả bao nhiêu dữ liệu, dữ liệu phân bố thế nào, statistics của database, cách join/sort và chi phí đọc dữ liệu thực tế.

---

# 4. Composite Index

## Bài nói

> Composite index là index gồm nhiều column. Ví dụ `(department_id, role_id)`.
>
> Thứ tự column quan trọng. Với B-Tree composite index, em nhớ quy tắc leftmost prefix: index thường dễ tận dụng khi query bắt đầu từ column bên trái của index.
>
> Ví dụ index `(department_id, role_id)`: query theo `department_id` có thể dùng tốt; query chỉ theo `role_id` thường không tận dụng index này theo cách thông thường.

### “Leftmost prefix” là gì?

> Nghĩa là với index nhiều cột, mình thường phải bắt đầu từ cột bên trái. Có thể hình dung index được sắp trước theo `department_id`, bên trong mỗi department mới tiếp tục theo `role_id`. Nếu bỏ qua `department_id` và chỉ tìm `role_id`, thứ tự đó không còn thuận lợi như khi tìm từ cột đầu.

### Chọn thứ tự column thế nào?

> Em nhìn query thực tế: thường filter cột nào trước, có equality/range condition gì và có `ORDER BY` hay không. Em không chọn chỉ dựa vào “cột nào unique hơn”, sau đó em kiểm tra lại bằng execution plan.

---

# 5. LIKE và function trên column

> Với B-Tree thông thường, tìm prefix như `LIKE 'abc%'` có thể tận dụng index trong điều kiện phù hợp vì database biết điểm bắt đầu của chuỗi.
>
> Còn `LIKE '%abc'` có wildcard ở đầu nên database không biết bắt đầu tìm từ đâu trong B-Tree theo cách thông thường. Nếu cần search văn bản phức tạp em cân nhắc full-text search, trigram hoặc search engine tùy bài toán.

> Tương tự, nếu query dùng `LOWER(name)` nhưng index chỉ nằm trên `name`, database có thể không tận dụng index đó theo cách mình mong muốn. Một số database hỗ trợ expression/functional index để index chính biểu thức đó.

### Trigram/full-text là gì?

> Nếu interviewer hỏi sâu em mới đi vào. Nói đơn giản: đó là những cơ chế index/search phù hợp hơn B-Tree cho bài toán tìm kiếm text, đặc biệt khi cần tìm từ hoặc chuỗi ở nhiều vị trí chứ không chỉ prefix.

---

# 6. Case thực tế: query chậm

## Bài nói 60 giây

> Khi gặp query chậm, đầu tiên em lấy đúng query đang chạy và đo thời gian. Sau đó em xem execution plan để biết database đang đọc dữ liệu và join như thế nào.
>
> Nếu thấy database đọc quá nhiều row vì thiếu index phù hợp thì em thiết kế index dựa trên điều kiện `WHERE`, `JOIN` hoặc `ORDER BY` thường dùng. Sau đó em chạy lại plan và benchmark để xem có thực sự nhanh hơn không.
>
> Em cũng kiểm tra ảnh hưởng ngược lại, vì thêm index sẽ làm insert/update tốn thêm chi phí. Nếu query chậm do nguyên nhân khác thì em không cố chữa bằng index.

### Query chậm có phải luôn do thiếu index?

> Không. Có thể application gọi query lặp lại quá nhiều, lấy quá nhiều field/row, join chưa hợp lý, database đang bị lock, connection pool hết connection hoặc network chậm.

### N+1 là gì nếu interviewer hỏi?

> Là trường hợp thay vì lấy dữ liệu bằng một hoặc vài query hợp lý, application chạy thêm một query cho từng item. Ví dụ lấy 100 orders rồi chạy 100 query riêng để lấy customer của từng order. Tổng số query tăng mạnh và làm request chậm.

### Benchmark là gì?

> Là đo performance trước và sau thay đổi dưới điều kiện tương đối giống nhau để biết thay đổi có thực sự tốt hơn hay không, thay vì chỉ cảm giác “có index chắc sẽ nhanh”.

## Cách nhớ

`Query chậm → đo → EXPLAIN → xem đọc bao nhiêu row → sửa query/index → đo lại → kiểm tra ảnh hưởng write`
