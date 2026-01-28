# Chapter 11: Scaling MySQL – Khi "ngon nghẻ" trở thành bài toán "đau đầu"

## Mở đầu – Tản mạn dev

Chào anh em,

Chắc hẳn ai làm backend cũng từng trải qua cái cảnh tượng "kinh hoàng" này: Dự án start-up ngày đầu traffic lèo tèo, database chạy một con VM ghẻ cũng "ngon lành cành đào". Bỗng một ngày đẹp trời, marketing chạy campaign hiệu quả, user đổ về ầm ầm như thác lũ. Con MySQL server tội nghiệp bắt đầu la ó, CPU chạm nóc 100%, query chậm như rùa bò, app thì xoay vòng vòng loading.

Lúc này, sếp thì réo, anh em thì cuống cuồng đi tuning query, đắp thêm RAM, thêm CPU (vertical scaling). Nhưng rồi cũng đến lúc tiền đắp vào phần cứng không còn hiệu quả nữa. Đó là lúc chúng ta phải ngồi lại nghiêm túc về câu chuyện **Scaling MySQL**.

Bài viết này mình sẽ chắt lọc những gì tinh túy nhất từ Chapter 11 của cuốn sách gối đầu giường "High Performance MySQL, 4th Edition", kết hợp với chút trải nghiệm "xương máu" của mình để anh em có cái nhìn thực tế nhất về việc scale database. Không lý thuyết suông đâu nhé, toàn đồ chơi xịn đấy.

## Tổng quan

Scaling (mở rộng) không đơn thuần là làm cho hệ thống to ra. Nó là khả năng xử lý traffic ngày càng tăng mà vẫn giữ được performance ổn định với chi phí chấp nhận được.

Trong MySQL, trước khi lao đầu vào các kỹ thuật cao siêu như Sharding, điều quan trọng nhất là anh em phải xác định được **Workload** của mình thuộc loại nào:
1.  **Read-Bound:** Hệ thống chủ yếu là đọc data (SELECT). Ví dụ: trang tin tức, e-commerce catalog.
2.  **Write-Bound:** Hệ thống ghi, sửa, xóa liên tục. Ví dụ: hệ thống logging, tracking user behavior, thanh toán.

Hiểu sai workload là đi bụi ngay từ bước thiết kế. (haizz). Dưới đây là 3 chiến thuật cốt lõi mà anh em cần nắm vững.

## 1. Scaling Reads: Đừng để con Source gánh còng lưng

Đây là bước đầu tiên và dễ ăn nhất. Nguyên lý rất đơn giản: **Một thằng ghi, nhiều thằng đọc**.

Chúng ta vẫn giữ một con Source (Master) để ghi data, nhưng tạo ra hàng loạt các con Replicas (Slaves) chỉ để phục vụ việc đọc. App sẽ trỏ các query SELECT về một "Read Pool" thay vì chọc thẳng vào Source.

Để làm việc này "ngon lành", anh em cần một con Load Balancer đứng giữa (như HAProxy). Nó giúp phân tải traffic đều cho các Replicas.

**Cấu hình HAProxy cơ bản để anh em hình dung:**

```haproxy
listen mysql-readpool
    bind 127.0.0.1:3306
    mode tcp
    option mysql-check user haproxy_check
    balance leastconn
    server mysql-1 10.0.0.1:3306 check
    server mysql-2 10.0.0.2:3306 check
```

**Lưu ý cực quan trọng:** Anh em nên dùng thuật toán `leastconn` (kết nối ít nhất) thay vì `round-robin`. Vì sao? Vì query MySQL có cái chạy nhanh, cái chạy chậm. Nếu chia đều theo lượt, có khi một con Replica đen đủi vớ phải toàn query nặng (heavy join) sẽ sập nguồn, trong khi mấy con kia ngồi chơi. `leastconn` sẽ giúp tránh vụ này.

## 2. Queuing: Vũ khí bí mật trước khi Sharding

Khi hệ thống bị nghẽn write (Write-Bound), anh em khoan hãy nghĩ tới Sharding vội. Sharding là con dao hai lưỡi, phức tạp và lằng nhằng khủng khiếp.

Thay vào đó, hãy kiểm tra xem **mọi write có cần phải đồng bộ (synchronous) ngay lập tức không?**

Ví dụ: Việc lưu log activity của user, hay cập nhật số lượt view. Có cần thiết user click cái là phải ghi vào DB ngay không? Hay trễ 5-10s cũng chả chết ai?

Giải pháp ở đây là dùng **Message Queue** (như Kafka, RabbitMQ). Đẩy write request vào queue, rồi có một worker túc tắc lấy ra ghi vào MySQL theo batch. Cách này giúp làm phẳng (flatten) các đỉnh traffic (spikes), giữ cho DB không bị shock khi traffic tăng đột biến. Nó biến các thao tác ghi ngẫu nhiên thành tuần tự và kiểm soát được tốc độ. Một chiêu đơn giản mà hiệu quả đến "ảo ma".

## 3. Scaling Writes: Sharding - Cuộc chơi của những gã khổng lồ

Nếu đã tuning query chán chê, dùng Queue rồi mà vẫn không chịu nổi nhiệt, thì chúc mừng, anh em đã đến level buộc phải **Sharding**.

Sharding hiểu nôm na là chia nhỏ data ra nhiều database server khác nhau. Có 2 hướng tiếp cận:

### Functional Sharding (Chia theo chức năng)
Cái này dễ làm trước. Tách các bảng không liên quan ra các cụm DB riêng.
- Cụm A: Chứa bảng `Users`, `Authen`.
- Cụm B: Chứa bảng `Orders`, `Products`.
- Cụm C: Chứa `Logs`, `Tracking`.

Cách này giúp cô lập lỗi. Cụm Log có chết thì user vẫn login mua hàng bình thường.

### Data Sharding (Chia nhỏ dữ liệu)
Đây mới là trùm cuối. Cùng một bảng `Users` nhưng user ID từ 1-1tr nằm ở Server A, từ 1tr-2tr nằm ở Server B.

Cái khó nhất ở đây là chọn **Partitioning Key** (Sharding Key). Chọn sai là "ăn hành" ngập mặt.
- Nếu chọn Shard key là `user_id`: Tìm data theo user thì cực nhanh (biết ngay nó nằm ở server nào). Nhưng muốn query "Lấy 10 đơn hàng mới nhất của toàn hệ thống" thì phải đi hỏi tất cả các shard (scatter-gather), performance sẽ cực tệ.

Ngày nay, để đỡ khổ khi làm sharding thủ công, anh em architect hay dùng các tool như **Vitess** hoặc **ProxySQL**. Bọn này đóng vai trò như một lớp proxy thông minh, giúp app nhìn vào tưởng như đang giao tiếp với một DB duy nhất, nhưng bên dưới nó tự định tuyến query sang đúng shard. (Ngon lành!).

## Các sai lầm thường gặp (Common Pitfalls)

1.  **Lạm dụng Health Check:** Cấu hình load balancer health check quá đơn giản (chỉ ping port 3306). DB sống nhưng đang bị lock hoặc replication lag cả tiếng đồng hồ mà vẫn đẩy traffic đọc vào -> User thấy dữ liệu cũ rích. (Sad).
2.  **Chọn sai Sharding Key:** Chọn key dựa trên giả định hiện tại mà không tính tới tương lai. Ví dụ shard theo `location`, lỡ đâu sau này user ở Hà Nội gấp 10 lần user ở Đà Nẵng thì một server sẽ quá tải (Hotspot), còn server kia thì mốc meo.
3.  **Cross-Shard Joins:** Cố gắng JOIN các bảng nằm trên các shard khác nhau. Đây là điều cấm kỵ. Khi đã shard, hãy chấp nhận xử lý việc tổng hợp dữ liệu ở tầng Application hoặc chấp nhận data denormalization (dư thừa dữ liệu).

## Kinh nghiệm thực tế / Lesson Learned

Hồi mình làm hệ thống cho một bên e-commerce, ban đầu lười không tách Read/Write. Đến đợt Flash Sale, lượng người xem hàng (Read) tăng đột biến làm lock table, dẫn đến người mua hàng (Write) không checkout được. Doanh thu tụt, sếp chửi té tát.

Sau đó tách Read Pool ra dùng HAProxy như mục 1. Nhưng lại dính chưởng **Replication Lag**. User vừa đặt hàng xong (ghi vào Master), reload trang danh sách đơn hàng (đọc từ Replica) thì chưa thấy đơn đâu. User tưởng lỗi đặt lại lần nữa -> Duplicate order.
-> **Bài học:** Với các flow quan trọng như thanh toán, sau khi write xong, bắt buộc các read tiếp theo của user đó phải đọc từ Master (hoặc dùng cơ chế Pinning session) để đảm bảo Consistency.

## Kết luận

Scaling MySQL là một hành trình, không phải đích đến.
- Bắt đầu từ việc hiểu rõ workload (Read hay Write bound).
- Tận dụng tối đa Read Replicas và Caching.
- Dùng Queue để giảm tải ghi.
- Chỉ Sharding khi không còn đường lùi.

Tư duy đúng là: **"Keep it simple until you can't"**. Đừng vẽ vời kiến trúc microservices hay sharding phức tạp khi traffic chưa tới mức đó.

Topic này còn rất rộng, anh em nào muốn đào sâu thì nên tìm hiểu tiếp về **Vitess**, **Orchestrator** (để quản lý failover) và **ProxySQL**.

Chúc anh em build được hệ thống "bùm" traffic vẫn chạy phà phà nhé!

**Nguồn:**
- High Performance MySQL, 4th Edition – Chapter 11: Scaling MySQL[1]
- Official MySQL Documentation