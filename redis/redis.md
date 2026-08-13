# Redis Interview Notes

## Redis là gì?
Redis là in-memory data store, thường dùng cho cache, session, rate limit, counters, queue nhẹ và distributed coordination. Điểm mạnh chính là tốc độ truy cập rất cao vì dữ liệu chủ yếu nằm trong RAM.

## Data types quan trọng
- String: cache value, token, counter.
- Hash: object nhỏ như user/session.
- List: queue đơn giản.
- Set: tập unique, membership.
- Sorted Set: leaderboard, ranking, delayed scheduling.
- Stream: event stream/consumer group.

## Cache-aside pattern
Luồng đọc phổ biến: app đọc Redis -> cache hit thì trả dữ liệu -> cache miss thì đọc DB -> ghi lại Redis với TTL -> trả kết quả.

```js
const cached = await redis.get(key);
if (cached) return JSON.parse(cached);
const data = await db.findById(id);
await redis.set(key, JSON.stringify(data), { EX: 300 });
return data;
```

## TTL
TTL giúp cache tự hết hạn.
- TTL ngắn: fresh hơn nhưng miss nhiều hơn.
- TTL dài: hit rate tốt hơn nhưng stale lâu hơn.

## Cache invalidation
Khi DB update, pattern thường gặp là update DB trước rồi delete cache key. Lần đọc tiếp theo sẽ cache lại.

```js
await db.updateUser(id, payload);
await redis.del(`user:${id}`);
```

## Cache stampede
Một key hot hết hạn, rất nhiều request cùng miss và cùng đánh DB.
Giải pháp: distributed lock, TTL jitter, stale-while-revalidate, refresh trước expiration, request coalescing.

## Cache penetration
Request liên tục hỏi dữ liệu không tồn tại, làm cache miss và đánh DB.
Giải pháp: negative cache, Bloom filter trong một số hệ thống, validate input, rate limit.

## Cache avalanche
Nhiều key hết hạn cùng lúc làm traffic đổ về DB.
Giải pháp: random TTL jitter, chia thời điểm refresh, circuit breaker/rate limit.

## Distributed lock
Ý tưởng cơ bản:
```text
SET lock:key unique-token NX PX 10000
```
Chỉ owner có đúng token mới được unlock. Không nên `DEL lock:key` mù quáng vì có thể xóa lock của request khác sau khi lock cũ hết hạn.

## Persistence
- RDB snapshot: snapshot định kỳ, compact, restore nhanh.
- AOF: log thao tác, durability tốt hơn tùy config nhưng tốn I/O hơn.
Có thể dùng kết hợp.

## Eviction policy
Khi hết memory có các nhóm policy như noeviction, allkeys-lru, allkeys-lfu, volatile-lru, volatile-lfu, random/ttl variants. Chọn theo workload.

## Redis có thay DB được không?
Không mặc định. RAM đắt và durability/query model khác RDBMS. Redis thường bổ trợ DB hơn là thay thế primary database.

## Rate limiting với Redis
Có thể dùng fixed window, sliding window, token bucket/leaky bucket. Redis phù hợp vì atomic increment và TTL.

## Hot key
Một key nhận quá nhiều traffic có thể thành bottleneck. Giải pháp tùy case: local cache, sharding key, read scaling, giảm contention, batch request.

## Câu hỏi phỏng vấn
### Redis nhanh vì sao?
Dữ liệu chủ yếu ở RAM, data structure tối ưu và network/event model hiệu quả.

### Khi nào không nên cache?
Dữ liệu thay đổi liên tục, khó invalidation, ít được đọc lại, hoặc cache cost lớn hơn lợi ích.

### Cache-aside nhược điểm gì?
Có thể stale data, cache miss đầu tiên chậm và dễ stampede nếu key hot hết hạn.

### TTL nên bao lâu?
Không có số cố định. Dựa vào freshness, read frequency, cost truy vấn nguồn và khả năng chịu stale.

### Redis lock cần lưu ý gì?
Unique owner token, timeout và unlock an toàn; phải hiểu failure assumptions.

## Cheat sheet
- Redis = in-memory, nhanh.
- Cache-aside: cache -> miss -> DB -> set cache.
- TTL dựa trên freshness và load.
- Stampede: lock/jitter/stale-while-revalidate.
- Penetration: negative cache/Bloom filter.
- Avalanche: tránh expiry đồng loạt.
- Distributed lock cần ownership token + timeout.
- Redis không mặc định thay primary database.
