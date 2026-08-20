# 10 — AWS S3 & Large File Upload

Mục tiêu: kể được một flow upload file lớn từ vấn đề thực tế, đồng thời giải thích được `presigned URL`, multipart upload và các failure case.

---

# 1. Case: upload file lớn qua backend

## 💬 Bài nói 60–90 giây

> Một case em từng gặp là upload file lớn qua backend. Khi file lên tới hàng trăm MB, request có thể gặp `413 Request Entity Too Large`, timeout hoặc làm application server tiêu tốn nhiều bandwidth và memory.
>
> Nếu flow là client → Nginx → Node.js → S3 thì toàn bộ bytes của file phải đi qua application server. Backend lúc đó trở thành middleman dù nó không cần xử lý nội dung file.
>
> Với file lớn, em ưu tiên để backend xác thực user, kiểm tra quyền upload rồi tạo **presigned URL** — tức là một URL tạm thời cho phép client upload trực tiếp lên S3 trong phạm vi mình cấp.
>
> Nhờ vậy data path đi thẳng từ client tới S3, backend vẫn kiểm soát quyền nhưng không phải proxy toàn bộ file.

---

# 🧾 Thuật ngữ

### **S3 Object** *(một file/object được lưu trong bucket)*

### **Bucket** *(container logic chứa các object trong S3)*

### **Presigned URL** *(URL có chữ ký tạm thời cho phép thực hiện một operation S3 cụ thể trong thời gian giới hạn)*

### **Data path** *(đường dữ liệu thực tế đi qua các thành phần nào)*

### **Control path** *(đường xử lý quyền/metadata/điều phối, không nhất thiết mang toàn bộ bytes)*

📌 Với direct upload:

```text
Control path:
Client → Backend → kiểm tra quyền → tạo presigned URL

Data path:
Client ─────────────────────────→ S3
```

---

# 2. Tại sao không chỉ tăng `client_max_body_size`?

## 💬 Cách nói

> Tăng Nginx limit có thể giải quyết 413 nếu mình vẫn muốn proxy upload qua server. Nhưng nếu kiến trúc không cần backend đọc toàn bộ file thì direct upload giúp bỏ bottleneck tốt hơn việc chỉ tăng limit ngày càng lớn.

### `413 Request Entity Too Large`

Nghĩa là request body lớn hơn limit mà một layer chấp nhận.

Có thể đến từ:

- Nginx;
- framework/server;
- upstream proxy khác.

⚠️ Không nên thấy 413 rồi chỉ sửa Nginx nếu còn layer khác có limit.

---

# 3. Presigned URL có an toàn không?

## 💬 Bài nói

> Presigned URL không phải public credential lâu dài. Nó chỉ cấp quyền tạm thời cho operation cụ thể. Trước khi tạo URL, backend vẫn phải authenticate user và authorize xem user có được upload object đó không.
>
> Em giới hạn thời gian hết hạn hợp lý và kiểm soát object key/path để user không tự chọn ghi đè file của người khác.

### **Authentication** *(xác định user là ai)*

### **Authorization** *(xác định user đó được phép làm gì)*

### **Expiration** *(thời điểm URL hết hiệu lực)*

⚠️ **Dễ bị bắt bẻ:**

> “Presigned URL an toàn vì nó expire.”

✅ **Cách nói an toàn:**

> Expiration chỉ là một lớp. Backend vẫn phải kiểm tra quyền, scope object key và validation cần thiết trước khi cấp URL.

---

# 4. Làm sao kiểm soát loại file và size?

Có vài lớp kiểm tra tùy flow:

- backend quyết định object key và metadata;
- client gửi thông tin file trước khi xin URL;
- có thể giới hạn content type/conditions tùy cơ chế ký;
- sau upload backend có thể đọc object metadata để verify;
- nếu file cần security scan thì xử lý sau upload trước khi mark active.

### **Metadata** *(thông tin mô tả object như size, content type, key...)*

⚠️ Không nên tin `Content-Type` từ client như bằng chứng tuyệt đối file an toàn.

---

# 5. Multipart Upload

## **Multipart upload** *(chia file lớn thành nhiều part rồi upload từng part, sau đó complete lại thành một object)*

## 📌 Ví dụ

```text
File 1 GB
  ↓
Part 1 ─┐
Part 2 ─┤
Part 3 ─┤ → S3
Part 4 ─┘
  ↓
Complete multipart upload
```

## Lợi ích

- một part lỗi chỉ retry part đó;
- có thể upload một số part song song;
- phù hợp hơn với file lớn/network không ổn định;
- có thể hỗ trợ resume theo flow của application.

### **Resume** *(tiếp tục từ phần chưa hoàn thành thay vì upload lại toàn bộ)*

⚠️ Cần cleanup multipart upload bị bỏ dở, nếu không có thể tồn tại các part chưa complete.

---

# 6. Upload xong nhưng DB chưa lưu metadata

Đây là case rất hay để interviewer đào.

## 📌 Tình huống

1. Client upload S3 thành công.
2. Trước khi gọi API confirm, app/browser crash.
3. Object tồn tại trên S3 nhưng DB không có record tương ứng.

## 💬 Cách xử lý

> Em không giả định S3 upload và DB insert là một transaction duy nhất vì chúng là hai hệ thống khác nhau. Em có thể upload vào trạng thái/key tạm, sau đó client confirm để backend verify object và tạo metadata record. Ngoài ra có cleanup job xóa những object tạm quá lâu không được finalize.

### **Orphan object** *(object tồn tại trên storage nhưng không còn record/business entity tương ứng)*

### **Finalize / Confirm** *(bước xác nhận upload hoàn thành và chuyển object sang trạng thái chính thức)*

---

# 7. Nếu DB lưu xong nhưng upload thất bại?

Có thể lưu record với state:

```text
pending_upload
uploaded
processing
active
failed
```

### **State machine** *(mô hình quy định entity có những trạng thái nào và được chuyển từ trạng thái nào sang trạng thái nào)*

Không nhất thiết cần framework state machine; ý chính là trạng thái phải rõ ràng để recover được.

---

# 8. Stream upload qua backend khi nào vẫn hợp lý?

Direct-to-S3 không phải luôn đúng.

Backend proxy/stream có thể cần nếu:

- phải transform/encrypt dữ liệu ở server;
- policy không cho client access storage trực tiếp;
- cần kiểm soát protocol đặc biệt;
- file nhỏ và simplicity quan trọng hơn.

Nếu proxy qua Node, nên stream thay vì buffer toàn bộ file nếu use case cho phép.

### **Buffer toàn bộ** *(đọc cả file vào memory trước khi xử lý)*

### **Streaming** *(xử lý file theo chunk khi dữ liệu đang đi qua)*

---

# 9. Timeout có những loại nào?

Upload qua nhiều layer có thể gặp:

- client timeout;
- Nginx/client body timeout;
- proxy read/send timeout;
- Node application timeout;
- storage/network timeout.

## 💬 Cách nói

> Khi debug timeout em xác định request dừng ở layer nào trước thay vì tăng tất cả timeout cùng lúc.

---

# 10. Security

Các điểm nên nhắc nếu interviewer hỏi:

- không để AWS secret key ở client;
- presigned URL có thời gian sống giới hạn;
- authorize object key;
- validate size/type theo mức cần thiết;
- tránh object name collision;
- malware scan nếu business cần;
- bucket permission không mở public ngoài ý muốn.

---

# 🎯 Interviewer hỏi tiếp

### Presigned URL bị lộ thì sao?

> Trong thời gian URL còn hiệu lực, người có URL có thể thực hiện operation được ký. Vì vậy em giữ expiration hợp lý và scope quyền/key nhỏ nhất cần thiết.

### Làm sao biết upload thực sự tồn tại?

> Backend có thể gọi `HEAD`/metadata check, dùng callback/confirm từ client hoặc event từ S3 tùy flow. Em không chỉ tin client nói “upload thành công”.

### Upload song song nhiều part có phải càng nhiều càng nhanh?

> Không. Concurrency quá cao có thể tốn bandwidth/memory hoặc làm client/network không ổn định. Em giới hạn và benchmark.

### Có transaction giữa S3 và database không?

> Không phải local ACID transaction thông thường. Em xử lý bằng state rõ ràng, retry, idempotency và cleanup/compensation.

---

# ⚠️ Dễ bị bắt bẻ

❌ “File lớn thì cứ tăng Nginx limit.”  
✅ “Đó là một option nếu vẫn proxy qua server; nếu backend không cần xử lý bytes thì direct S3 thường giảm tải tốt hơn.”

❌ “Presigned URL nghĩa là client có AWS credential.”  
✅ “Client nhận quyền tạm thời cho operation đã ký, không nhận secret key của server.”

❌ “Multipart upload đảm bảo resume tự động.”  
✅ “Multipart tạo nền tảng để retry/resume theo part; application vẫn cần quản lý upload ID/part state nếu muốn resume.”

---

# 📌 STAR version

**Situation:** File lớn đi qua Node/Nginx gặp size limit và timeout.  
**Task:** Cho phép upload ổn định mà không làm application server gánh toàn bộ bytes.  
**Action:** Kiểm tra proxy limits, chuyển sang presigned direct upload khi phù hợp, multipart cho file lớn, verify/finalize sau upload.  
**Result:** Giảm tải server và retry/resume dễ hơn.

# Cách nhớ

**Client xin quyền → backend authorize → presigned URL → upload thẳng S3 → verify/finalize → cleanup object lỗi/bỏ dở**