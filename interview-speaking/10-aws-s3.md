# 10 — AWS S3 & Large File Upload

# Case: upload file lớn

## Bài nói

> Một case em từng gặp là upload file lớn qua backend. Khi file khoảng hàng trăm MB, request có thể gặp `413 Request Entity Too Large`, timeout hoặc làm application server tiêu tốn nhiều memory/bandwidth.
>
> Nếu flow là client → Node server → S3 thì backend trở thành middleman cho toàn bộ bytes. Với file lớn em ưu tiên để backend authenticate/authorize rồi tạo presigned URL, sau đó client upload trực tiếp lên S3.
>
> Như vậy backend vẫn kiểm soát quyền upload nhưng data path không phải đi qua Node server. Với file rất lớn hoặc network không ổn định, multipart upload giúp chia file thành nhiều part và retry riêng từng part.

---

# Interviewer đào sâu

### Presigned URL có an toàn không?

> URL có quyền tạm thời nên cần expiration ngắn hợp lý, giới hạn operation/object key và validate user trước khi cấp URL. Không coi URL là public credential lâu dài.

### Làm sao biết upload đã xong?

> Tùy flow, client có thể gọi API complete/confirm sau upload, backend verify object metadata/head request, hoặc dùng S3 event để trigger processing.

### Nếu upload qua Nginx bị 413?

> Kiểm tra `client_max_body_size` và các proxy/body timeout. Nhưng nếu kiến trúc cho phép direct-to-S3 thì em ưu tiên giảm việc proxy large payload qua application server thay vì chỉ tăng limit mãi.

### Multipart upload lợi gì?

> Retry từng part, upload song song có kiểm soát và resume tốt hơn so với phải upload lại toàn bộ file.

### File upload xong nhưng DB chưa lưu metadata?

> Đây là distributed consistency problem. Có thể upload vào temporary key/status rồi confirm/finalize, có cleanup job cho orphan objects và thiết kế operation idempotent.

---

# STAR version

**Situation:** File lớn upload qua application/Nginx gặp size limit và timeout.

**Action:** Kiểm tra proxy limits nhưng chuyển data path sang presigned direct upload khi phù hợp; dùng multipart cho file lớn.

**Result:** Giảm tải application server và làm retry/resume dễ hơn.

## Cách nhớ

`Client → backend auth → presigned URL → direct S3 → verify → cleanup`
