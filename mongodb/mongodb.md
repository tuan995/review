# MongoDB Interview Notes

## MongoDB là gì?
MongoDB là document database lưu BSON documents. Nó phù hợp khi schema cần linh hoạt và dữ liệu thường được đọc theo aggregate/document.

## Thiết kế theo access pattern
Không normalize mọi thứ theo thói quen của SQL. Hỏi trước: dữ liệu nào thường được đọc và cập nhật cùng nhau?

## Embed vs Reference
### Embed
Dùng khi dữ liệu con nhỏ, cùng vòng đời và thường đọc cùng parent.

Ưu điểm: ít round trip, atomic update trong một document.
Nhược điểm: document có thể phình lớn và duplicate dữ liệu nếu thiết kế không phù hợp.

### Reference
Dùng khi dữ liệu lớn, shared hoặc có vòng đời độc lập.

Ưu điểm: giảm duplication.
Nhược điểm: có thể cần thêm query hoặc lookup.

## Index
MongoDB hỗ trợ single-field, compound, multikey, text và geospatial index. Compound index phụ thuộc thứ tự field và query pattern.

## ESR heuristic
Một heuristic phổ biến cho compound index:
- Equality
- Sort
- Range

Vẫn phải kiểm tra bằng query plan trên workload thật.

## explain
Dùng `explain()` để xem query planner và execution statistics.

Chú ý:
- COLLSCAN.
- IXSCAN.
- documents examined.
- documents returned.

Nếu examined rất lớn nhưng returned ít thì query/index có thể chưa tốt.

## Aggregation Pipeline
Stage thường gặp:
- `$match`
- `$project`
- `$group`
- `$sort`
- `$lookup`
- `$unwind`

Đẩy `$match` sớm thường giúp giảm dữ liệu phải xử lý.

## Replica Set
Replica Set phục vụ high availability và redundancy. Thường có primary và secondary; secondary replicate dữ liệu từ primary và có thể election primary mới khi cần.

## Sharding
Sharding chia dữ liệu ngang để scale dataset và throughput.

Shard key tốt cần:
- phân phối dữ liệu tương đối đều.
- tránh hotspot.
- phù hợp query pattern.
- cardinality đủ tốt.

## Replica Set vs Sharding
Replica Set = replication/availability.
Sharding = horizontal partitioning/scale.

## Transaction
MongoDB hỗ trợ multi-document transaction, nhưng nếu dữ liệu luôn cần atomic cùng nhau thì nên cân nhắc document model/embedding trước.

## Câu hỏi phỏng vấn
### Khi nào chọn MongoDB?
Khi document model phù hợp access pattern, schema cần linh hoạt và workload không phụ thuộc mạnh vào relational joins/constraints.

### Embed hay reference?
Embed khi nhỏ, cùng vòng đời, thường đọc cùng nhau; reference khi lớn, shared hoặc độc lập.

### Vì sao shard key quan trọng?
Nó quyết định cách phân phối dữ liệu và traffic; shard key xấu có thể gây hotspot.

### Replica Set khác Sharding thế nào?
Replica Set chủ yếu cho HA; Sharding chủ yếu cho horizontal scale.

## Cheat sheet
- Thiết kế theo access pattern.
- Embed cho aggregate nhỏ/cùng vòng đời.
- Reference cho shared/lớn/độc lập.
- Compound index phụ thuộc thứ tự field.
- `explain()` để kiểm tra IXSCAN/COLLSCAN.
- Replica Set = HA.
- Sharding = scale.
- Shard key quyết định phân phối dữ liệu và traffic.
