# 08 — Background Job, Cron & Queue

Mục tiêu: hiểu khi nào dùng cron, khi nào dùng queue, và giải thích được retry/idempotency mà không phụ thuộc vào jargon.

---

# 1. Background job là gì?

## **Background job** *(công việc được xử lý ngoài request-response hiện tại)*

## 💬 Bài nói

> Em đưa công việc ra background khi task chạy lâu, không cần trả kết quả ngay cho user hoặc cần retry độc lập. Ví dụ đồng bộ dữ liệu nhiều shop, gửi email, xử lý file hoặc job kiểm tra dữ liệu định kỳ.
>
> Request có thể tạo job rồi trả response sớm. Worker xử lý phía sau. Nhờ vậy API không bắt user chờ toàn bộ task dài hoàn thành.

### **Worker** *(process/service lấy job ra và thực hiện công việc)*

---

# 2. Cron là gì?

## **Cron / Scheduled job** *(job chạy theo lịch định trước)*

📌 Ví dụ:

- mỗi ngày 3 giờ sáng cleanup data;
- mỗi 10 phút kiểm tra trạng thái;
- mỗi ngày reconciliation dữ liệu Shopify.

## 💬 Cách nói

> Cron phù hợp khi trigger của công việc là thời gian. Ví dụ cứ mỗi ngày em cần chạy một job kiểm tra dữ liệu.

---

# 3. Queue là gì?

## **Queue** *(hàng đợi lưu task để worker lấy xử lý)*

📌 Ví dụ:

```text
API / Cron
    ↓ tạo job
Queue
    ↓
Worker A
Worker B
Worker C
```

Queue hữu ích khi cần:

- retry;
- chia workload cho nhiều worker;
- giới hạn concurrency;
- biết job nào pending/failed;
- recover sau process restart.

---

# 4. Cron vs Queue

## 💬 Bài nói

> Cron và queue không phải hai thứ thay thế hoàn toàn cho nhau. Cron quyết định **khi nào bắt đầu**, còn queue giúp quản lý **cách các task được xử lý**.
>
> Ví dụ cron chạy mỗi ngày nhưng thay vì cron tự loop 10.000 shop trong một process, cron chỉ tạo job vào queue. Worker lấy job dần với concurrency phù hợp.

## 📌 Flow

```text
Cron 03:00
   ↓
Tạo 10.000 job
   ↓
Queue
   ↓
Workers xử lý có giới hạn
```

---

# 5. Tại sao không để cron xử lý tất cả trực tiếp?

Nếu process chết giữa 10.000 item:

- khó biết item nào đã xong;
- task đang nằm trong memory bị mất;
- retry toàn bộ có thể tạo duplicate;
- khó chia workload cho nhiều worker.

Queue giúp tách từng task thành đơn vị xử lý rõ ràng hơn.

### **Persistent queue** *(queue lưu job ở storage để process restart job vẫn còn)*

---

# 6. Duplicate cron khi chạy nhiều instance

## 📌 Vấn đề

Application scale thành 3 PM2 instances/container:

```text
Instance 1 → cron 10:00
Instance 2 → cron 10:00
Instance 3 → cron 10:00
```

Một lịch có thể chạy 3 lần.

## 💬 Bài nói

> Khi deploy nhiều instance, em không giả định cron chỉ chạy một lần. Nếu mỗi instance đều register scheduler thì cùng job có thể chạy lặp.
>
> Với job cần một scheduler duy nhất, em có thể tách scheduler riêng, dùng queue unique job hoặc dùng distributed lock tùy architecture. Đồng thời bản thân job vẫn nên chịu được duplicate nếu có thể.

### **Distributed lock** *(lock được nhiều process/server cùng nhìn thấy để chỉ một bên sở hữu tại một thời điểm)*

### **TTL của lock** *(thời gian lock tự hết hạn để tránh process chết làm lock tồn tại mãi)*

⚠️ Distributed lock implementation có nhiều edge case. Không nên chỉ nói “dùng Redis lock là xong”.

---

# 7. Retry & Idempotency

## 💬 Bài nói

> Background job phải giả định có thể chạy lại. Ví dụ worker đã update database thành công nhưng crash trước khi đánh dấu job completed. Queue có thể giao lại job đó.
>
> Vì vậy operation quan trọng nên **idempotent** — tức là chạy lại cùng task không tạo dữ liệu sai hoặc nhân đôi.

📌 Ví dụ:

- update product theo Shopify ID → chạy lại vẫn update cùng record;
- create payout → cần business key/unique constraint để không tạo hai payout.

### **Acknowledge / ACK** *(worker báo cho queue rằng job đã xử lý xong)*

Nếu worker chết trước ACK, nhiều queue có thể redeliver job.

---

# 8. At-least-once là gì?

## **At-least-once delivery** *(job có thể được giao một hoặc nhiều lần; hệ thống ưu tiên không làm mất job)*

## 💬 Cách nói dễ hiểu

> Em thường thiết kế theo hướng queue có thể giao lại job. Vì vậy handler phải chịu được duplicate thay vì giả định mỗi job chắc chắn chỉ chạy đúng một lần.

⚠️ “Exactly once” là thuật ngữ dễ bị đào sâu. Nếu không cần, không nên mở đầu bằng nó.

---

# 9. Dead-letter Queue

## **Dead-letter queue / DLQ** *(nơi chứa những job đã fail quá số lần retry để không retry vô hạn)*

📌 Ví dụ:

```text
Job fail
 ↓ retry 1
fail
 ↓ retry 2
fail
 ↓ retry 3
fail
 ↓
DLQ / failed state
```

Sau đó team có thể inspect và reprocess thủ công/tự động tùy hệ thống.

---

# 10. Queue backlog

## **Backlog / Queue depth** *(số job đang chờ chưa được xử lý)*

Nếu job được tạo 1.000/phút nhưng worker chỉ xử lý 500/phút thì backlog tăng 500/phút.

Đây là dấu hiệu hệ thống đang tạo work nhanh hơn khả năng xử lý.

### **Oldest job age** *(job lâu nhất đã chờ bao lâu)*

Chỉ nhìn queue length chưa đủ. 1.000 job xử lý trong 10 giây khác với 1.000 job chờ 2 giờ.

---

# 11. Monitoring

Em muốn theo dõi:

- queue depth;
- oldest job age;
- processing time;
- success/failure rate;
- retry count;
- worker concurrency;
- external API/database error.

### **Observability** *(khả năng nhìn vào logs/metrics/traces để hiểu hệ thống đang xảy ra gì)*

---

# 🎯 Interviewer hỏi tiếp

### Queue có đảm bảo thứ tự không?

> Phụ thuộc queue và cách dùng. Nếu business cần ordering thì phải thiết kế rõ partition/key/concurrency. Em không giả định mọi queue luôn xử lý đúng thứ tự global.

### Nếu hai worker lấy cùng job thì sao?

> Queue thường có cơ chế visibility/claim/ack, nhưng distributed system vẫn cần handler idempotent vì redelivery hoặc failure có thể xảy ra.

### Cron có cần idempotent không?

> Nên có nếu operation có khả năng chạy lại. Deploy nhiều instance, retry hoặc manual rerun đều có thể làm cùng job chạy hơn một lần.

### Tại sao không retry vô hạn?

> Nếu lỗi permanent, retry vô hạn chỉ tốn resource và làm backlog tăng. Em giới hạn retry rồi chuyển failed/DLQ để inspect.

---

# ⚠️ Dễ bị bắt bẻ

❌ “Queue đảm bảo job chạy đúng một lần.”  
✅ “Em thiết kế handler chịu được việc job có thể được giao lại.”

❌ “Cron chỉ chạy một lần.”  
✅ “Một schedule có thể chạy trên nhiều instance nếu architecture không kiểm soát.”

❌ “Có queue thì không mất job.”  
✅ “Queue bền vững giúp giảm rủi ro mất job, nhưng guarantee cụ thể phụ thuộc queue/configuration và cách ACK.”

---

# 📌 Cách nhớ

**Thời gian trigger → cron → tạo job → queue → worker → concurrency → retry → idempotency → failed/DLQ → metrics**