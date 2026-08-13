# PostgreSQL Interview Notes

## PostgreSQL là gì?
PostgreSQL là hệ quản trị cơ sở dữ liệu quan hệ mã nguồn mở, mạnh về tính đúng đắn dữ liệu, transaction, extensibility và SQL nâng cao.

## MVCC
MVCC giúp nhiều transaction đọc/ghi đồng thời mà giảm blocking bằng cách duy trì nhiều version của row.

Ý chính:
- Reader thường không chặn writer theo kiểu lock toàn cục.
- Transaction thấy snapshot phù hợp với isolation level.
- Row cũ cần được cleanup sau này.

## VACUUM
Update/Delete trong PostgreSQL có thể để lại dead tuples do MVCC. VACUUM giúp thu hồi không gian để tái sử dụng và duy trì health của table/index.

`autovacuum` rất quan trọng trong production.

## ANALYZE
ANALYZE cập nhật statistics để optimizer ước lượng số dòng và chọn execution plan tốt hơn.

## Index types
### B-tree
Mặc định và phổ biến nhất. Tốt cho equality, range, sorting và prefix phù hợp.

### GIN
Phù hợp với dữ liệu chứa nhiều phần tử như array, full-text search và nhiều trường hợp JSONB containment.

### GiST
Framework index linh hoạt cho geometric, range, full-text và extension-specific use cases.

### BRIN
Phù hợp với bảng rất lớn khi dữ liệu có tương quan tốt với thứ tự vật lý, ví dụ timestamp tăng dần.

## JSONB
`JSONB` lưu JSON dạng binary để query/index hiệu quả hơn so với JSON text thông thường.

Dùng khi schema có phần linh hoạt nhưng vẫn muốn nằm trong PostgreSQL.

Không nên biến mọi dữ liệu quan hệ thành JSONB nếu cần join, constraint và query quan hệ rõ ràng.

## Primary key và unique constraint
Primary key đảm bảo unique + not null. Unique constraint đảm bảo uniqueness theo semantics của PostgreSQL.

## Foreign key
Giữ referential integrity giữa bảng nhưng cũng có chi phí khi write. Cột foreign key ở bảng con thường nên cân nhắc index nếu hay join/filter hoặc delete/update parent.

## Transaction isolation
PostgreSQL hỗ trợ các isolation level tiêu chuẩn; implementation thực tế dựa trên MVCC. `Read Uncommitted` được xử lý như `Read Committed`.

## Connection pool
Mỗi PostgreSQL connection có chi phí. Backend thường dùng connection pool và giới hạn hợp lý thay vì mở connection mới cho mỗi request.

## Long-running transaction
Transaction mở quá lâu có thể giữ snapshot cũ, cản cleanup dead tuples và làm bloat tăng.

## EXPLAIN ANALYZE
Dùng để xem plan và số liệu runtime thực tế.

Khi đọc plan, chú ý:
- estimated rows vs actual rows.
- scan type.
- join type.
- loops.
- sort/hash.
- execution time.

## Câu hỏi phỏng vấn

### MVCC để làm gì?
Cho phép concurrency tốt hơn bằng row versions và snapshot, giảm blocking giữa readers/writers.

### Vì sao cần VACUUM?
Để xử lý dead tuples do MVCC và duy trì khả năng tái sử dụng không gian/health của DB.

### B-tree khác GIN thế nào?
B-tree tốt cho scalar equality/range/sort; GIN phù hợp dữ liệu có nhiều token/elements như array, full-text, JSONB containment.

### Khi nào dùng JSONB?
Khi một phần schema cần linh hoạt nhưng vẫn cần transaction và query trong PostgreSQL.

### Vì sao connection pool quan trọng?
Giới hạn số connection, giảm overhead và bảo vệ DB khỏi quá tải.

## Cheat sheet
- PostgreSQL = relational + transaction mạnh.
- MVCC = row versions + snapshots.
- VACUUM cleanup dead tuples.
- ANALYZE cập nhật statistics.
- B-tree cho equality/range.
- GIN cho array/full-text/JSONB use cases.
- BRIN tốt với bảng lớn và dữ liệu tương quan theo thứ tự.
- Tránh transaction mở quá lâu.
- Dùng connection pool.
