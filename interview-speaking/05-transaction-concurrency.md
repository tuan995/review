# 05 — Transaction & Database Concurrency

> Mục tiêu của file này: không chỉ nhớ tên ACID/locking mà phải giải thích được bằng ngôn ngữ đơn giản khi interviewer hỏi ngược lại "ý em là gì?".

# 1. Transaction là gì?

## Bài nói đầy đủ

> Theo cách em hiểu, transaction dùng khi nhiều thao tác database thuộc cùng một nghiệp vụ và em không muốn hệ thống chỉ làm thành công một nửa.
>
> Ví dụ khi tạo order, hệ thống có thể phải tạo bản ghi `Order`, tạo các `LineItem`, rồi cập nhật một số dữ liệu liên quan. Nếu tạo Order thành công nhưng tạo LineItem thất bại, database sẽ có một order không đầy đủ.
>
> Khi đặt các thao tác đó trong transaction, nếu tất cả thành công thì em `commit`, tức là xác nhận lưu toàn bộ thay đổi. Nếu một bước quan trọng thất bại thì em `rollback`, tức là hủy những thay đổi của transaction đó để database không bị rơi vào trạng thái làm dở.
>
> Tuy nhiên transaction không nên mở quá lâu vì trong thời gian đó nó giữ database connection và có thể giữ lock. Vì vậy em cố giữ transaction ngắn và thường tránh gọi Shopify, Stripe hoặc external API trong lúc transaction đang mở.

### Commit là gì?

> Commit là thời điểm transaction xác nhận các thay đổi thành công. Sau commit, những thay đổi đó trở thành kết quả chính thức của transaction.

### Rollback là gì?

> Rollback là hủy các thay đổi chưa commit của transaction khi xảy ra lỗi hoặc khi mình chủ động quyết định không tiếp tục.

---

# 2. ACID — giải thích bằng ví dụ

ACID là bốn đặc tính thường được dùng để mô tả transaction. Khi phỏng vấn không nên chỉ đọc bốn chữ; hãy giải thích từng chữ.

## Atomicity — hoặc tất cả, hoặc không có gì

> Ví dụ em cần tạo Order và 3 LineItem. Nếu tạo đến LineItem thứ hai thì lỗi, Atomicity giúp em không để lại một nửa dữ liệu của nghiệp vụ đó. Transaction có thể rollback để các thay đổi thuộc transaction không được commit.

## Consistency — dữ liệu sau transaction vẫn đúng các rule

> Em hiểu Consistency đơn giản là trước và sau transaction, dữ liệu phải tiếp tục thỏa những rule mà hệ thống/database yêu cầu.
>
> Ví dụ `email` có unique constraint thì sau transaction không thể có hai record vi phạm constraint đó. Hoặc nếu hệ thống có rule số lượng tồn kho không được âm thì logic cập nhật phải bảo vệ rule đó.

### "Valid state" nghĩa là gì?

> Nếu dùng từ valid state, em muốn nói một trạng thái dữ liệu vẫn thỏa các constraint và rule của hệ thống. Nhưng khi phỏng vấn em ưu tiên nói thẳng rule cụ thể thay vì chỉ nói "valid state".

## Isolation — transaction chạy đồng thời ảnh hưởng nhau đến mức nào

> Ví dụ hai request cùng sửa một sản phẩm. Isolation liên quan đến việc một transaction được nhìn thấy thay đổi của transaction khác ở thời điểm nào và database ngăn các transaction concurrent gây kết quả sai ra sao. Mức bảo vệ này phụ thuộc isolation level.

## Durability — commit thành công thì dữ liệu phải được lưu bền vững

> Sau khi database báo transaction commit thành công, hệ thống database phải đảm bảo dữ liệu tồn tại theo durability guarantee của nó, kể cả khi sau đó process application restart.

---

# 3. Race Condition

## Race condition là gì?

> Race condition xảy ra khi kết quả phụ thuộc vào thứ tự hoặc thời điểm nhiều request/task chạy đồng thời. Mỗi request nhìn riêng có thể đúng, nhưng khi chúng xen kẽ nhau thì kết quả cuối có thể sai.

## Ví dụ stock = 1

> Giả sử stock hiện tại bằng 1. Request A đọc thấy 1. Gần như cùng lúc request B cũng đọc thấy 1. Cả hai đều kiểm tra `stock > 0` và đều cho phép mua. Sau đó cả hai cùng update.
>
> Vấn đề ở đây là đoạn "đọc → kiểm tra → ghi" gồm nhiều bước riêng biệt. Việc `if (stock > 0)` trong Node.js không đủ bảo vệ vì một process/request khác có thể thay đổi dữ liệu giữa các bước.

## Có thể xử lý thế nào?

> Một cách là atomic update có condition, ví dụ chỉ decrement khi stock vẫn lớn hơn 0. Một cách khác là transaction kết hợp row lock. Ngoài ra có optimistic concurrency nếu conflict không thường xuyên.
>
> Em chọn cách nào dựa vào mức độ conflict, yêu cầu correctness và database đang dùng, chứ không mặc định lock mọi thứ.

---

# 4. Optimistic và Pessimistic Locking

## Optimistic locking là gì?

> Optimistic nghĩa là mình giả định phần lớn thời gian các request sẽ không conflict. Record có thể có `version` hoặc một giá trị dùng để kiểm tra. Khi update, mình chỉ update nếu version vẫn giống lúc đọc. Nếu người khác đã sửa trước thì update thất bại và mình có thể đọc lại/retry.

**Ví dụ:** A và B cùng đọc `version = 5`. A update thành công và tăng lên 6. B cố update với điều kiện `version = 5` thì không còn match, nên B biết dữ liệu đã thay đổi.

## Pessimistic locking là gì?

> Pessimistic nghĩa là khi em chuẩn bị sửa dữ liệu, em lock record để transaction khác không thể thực hiện thao tác xung đột theo cách tương tự cho đến khi lock được giải phóng.
>
> Cách này hữu ích khi conflict có khả năng cao hoặc nghiệp vụ cần serialize access, nhưng nếu giữ lock lâu thì request khác phải chờ và throughput có thể giảm.

### Chọn cái nào?

> Conflict ít thì optimistic thường giúp concurrency tốt hơn. Conflict cao hoặc nghiệp vụ rất nhạy cảm có thể cần pessimistic/row locking. Em vẫn phải xem cụ thể database và query pattern.

---

# 5. Isolation Level và các lỗi thường gặp

## Isolation level là gì?

> Isolation level là mức độ database tách các transaction chạy đồng thời khỏi nhau. Mức isolation cao hơn thường ngăn được nhiều hiện tượng concurrency hơn, nhưng có thể đổi lại bằng nhiều waiting, locking hoặc chi phí xử lý hơn tùy database.

### Dirty read

> Transaction A sửa dữ liệu nhưng chưa commit. Transaction B đã đọc được giá trị chưa commit đó. Sau đó A rollback, nghĩa là B đã từng đọc một giá trị cuối cùng không tồn tại.

### Non-repeatable read

> Trong cùng transaction, em đọc cùng một row hai lần nhưng nhận hai giá trị khác nhau vì transaction khác đã commit update ở giữa hai lần đọc.

### Phantom read

> Em chạy cùng một query theo điều kiện hai lần nhưng lần sau xuất hiện hoặc biến mất thêm row do transaction khác insert/delete và commit ở giữa.

### Có phải luôn chọn isolation cao nhất?

> Không. Em chọn theo yêu cầu nghiệp vụ và đặc tính database. Nếu tăng isolation không cần thiết thì có thể giảm concurrency hoặc tăng contention.

---

# 6. Deadlock

## Deadlock là gì?

> Deadlock là tình huống hai transaction chờ tài nguyên của nhau và không transaction nào tự đi tiếp được.

**Ví dụ:**

```text
Transaction A: lock Order 1 → chờ Order 2
Transaction B: lock Order 2 → chờ Order 1
```

> Database thường phát hiện vòng chờ này và abort một transaction để phá deadlock.

## Em giảm deadlock thế nào?

> Em giữ transaction ngắn, truy cập các resource theo cùng một thứ tự khi có thể, đảm bảo query/index hợp lý để tránh lock phạm vi không cần thiết, và xử lý retry khi database báo deadlock nếu operation đó an toàn để retry.

---

# 7. External API + Database Transaction

### Tại sao không nên mở DB transaction rồi gọi Stripe/Shopify?

> External API là network call nên latency khó đoán và có thể timeout. Nếu em mở transaction, giữ connection/lock rồi chờ Stripe vài giây thì database resource cũng bị giữ trong vài giây đó. Khi nhiều request cùng làm vậy, connection pool và lock contention có thể trở thành bottleneck.

### Vậy workflow nhiều hệ thống xử lý thế nào?

> Em thường chia workflow thành các bước có trạng thái rõ ràng. Local database transaction chỉ bảo vệ dữ liệu trong database của mình. Với external service, em dùng webhook, retry và idempotency để đồng bộ trạng thái. Với workflow phức tạp hơn có thể cân nhắc outbox/event hoặc compensation.

### Idempotency là gì?

> Idempotency nghĩa là cùng một operation bị gửi lại nhiều lần nhưng hệ thống không tạo ra kết quả phụ ngoài ý muốn nhiều lần. Ví dụ Stripe gửi lại cùng một webhook thì mình không tạo hai payment record cho cùng một event.

### Compensation là gì?

> Vì không thể rollback một external service bằng DB rollback, đôi khi mình cần một hành động nghiệp vụ để "bù" cho bước đã thành công. Ví dụ bước sau thất bại thì tạo một operation hoàn tiền/hủy tương ứng thay vì giả vờ rằng transaction database có thể rollback cả hệ thống bên ngoài.

---

# 8. Câu hỏi interviewer có thể đào

### Tại sao `if` trong application không đủ chống race condition?

> Vì check và update không nhất thiết là một thao tác atomic. Request khác có thể thay đổi dữ liệu giữa lúc mình đọc và lúc mình ghi.

### Atomic operation là gì?

> Là operation được quan sát như một thao tác không bị chia nhỏ theo góc nhìn concurrency cần bảo vệ. Ví dụ một câu update có condition có thể vừa kiểm tra điều kiện vừa thay đổi giá trị trong một database operation, thay vì application đọc trước rồi update sau.

### Lock là gì?

> Lock là cơ chế database dùng để kiểm soát nhiều transaction cùng truy cập resource có xung đột. Tùy loại lock, transaction khác có thể vẫn đọc được hoặc phải chờ khi muốn ghi.

### Connection pool là gì?

> Application không mở một database connection mới cho từng query rồi đóng ngay. Nó thường giữ một nhóm connection để tái sử dụng. Nếu tất cả connection đang bận, query mới phải chờ; chờ quá lâu có thể timeout.

## Cách nhớ bằng câu chuyện

`Nhiều bước của một nghiệp vụ → transaction → commit/rollback → nhiều request cùng sửa → race condition → atomic update/locking/version → giữ transaction ngắn → external API dùng state + retry + idempotency`
