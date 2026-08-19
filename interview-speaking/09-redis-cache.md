# 09 — Redis & Caching

# 1. Khi nào dùng Redis?

## Bài nói

> Em dùng Redis khi cần dữ liệu truy cập nhanh và có thể lưu dưới dạng key-value với TTL, ví dụ cache, temporary state, rate limiting hoặc coordination tùy hệ thống.
>
> Nhưng em không cache chỉ vì Redis nhanh. Trước tiên em xác định bottleneck có thực sự nằm ở database/external API hay không và dữ liệu có chấp nhận stale không.

---

# 2. Cache-aside

## Bài nói

> Pattern em thường dùng là cache-aside. Application đọc Redis trước. Nếu cache miss thì đọc database, sau đó ghi kết quả vào Redis với TTL. Khi data thay đổi, application update source of truth và invalidate hoặc update cache.

```text
GET
 |
Redis ---- hit ---> return
 |
miss
 v
Database → set cache → return
```

### Cache invalidation?

> Đây là phần khó. Em có thể delete key khi write, update cache sau write hoặc dùng TTL làm safety net. Strategy phụ thuộc mức freshness business yêu cầu.

---

# 3. Cache stampede

> Nếu một hot key expire và hàng trăm request cùng miss, tất cả có thể query DB cùng lúc. Có thể dùng lock/single-flight, stale-while-revalidate hoặc randomize TTL để giảm stampede.

### TTL bao nhiêu?

> Không có số cố định. Dựa vào freshness requirement, read frequency và cost để rebuild data.

---

# 4. Redis down thì sao?

> Nếu Redis chỉ là cache, em cố thiết kế để application có thể fallback về database, đồng thời cần tránh fallback traffic làm DB overload. Nếu Redis đang giữ state quan trọng thì architecture và persistence/HA requirements sẽ khác.

---

# 5. Cache consistency

> Database thường là source of truth. Cache có thể stale trong một khoảng thời gian. Khi consistency rất quan trọng em không dựa vào cache như nơi duy nhất quyết định business invariant.

## Cách nhớ

`Need cache? → cache-aside → TTL → invalidation → stampede → fallback`
