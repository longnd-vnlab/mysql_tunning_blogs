# [MySQL Replication: Deep Dive & Best Practices từ dân Architect]

## Mở đầu – Tản mạn dev

Chắc hẳn anh em làm backend ai cũng từng trải qua cái cảm giác "lạnh sống lưng" khi production lăn đùng ra chết, hoặc đơn giản là khách hàng la ó vì query chậm như rùa bò. Lúc đó, giải pháp đầu tiên mà mọi người thường nghĩ đến để cứu vãn tình thế là: "Hay là dựng thêm con Slave để gánh read?".

Đúng, Replication là một trong những tính năng "thần thánh" nhất của MySQL giúp chúng ta scale-out hệ thống. Nhưng đời không như mơ, việc config một hệ thống replication chạy ngon, không lag, và quan trọng nhất là **crash-safe** (chết không mất dữ liệu) lại là một câu chuyện hoàn toàn khác. Mình từng thấy nhiều team set up replication xong để đó, đến khi Master sập thì Slave cũng lỗi luôn do thiếu log, hoặc dữ liệu trên Slave lệch pha cả cây số so với Master.

Bài viết này mình sẽ mổ xẻ Chapter 9 của cuốn sách "gối đầu giường" *High Performance MySQL (4th Edition)* để chia sẻ với mọi người những góc nhìn sâu hơn về Replication, không chỉ là "chạy được" mà là "chạy ngon".

## Tổng quan

Về cơ bản, Replication giải quyết bài toán đồng bộ dữ liệu giữa các instance. Master (Source) ghi log các thay đổi, và Slave (Replica) replay lại các log đó. Nghe đơn giản ha? Nhưng nó là nền tảng cốt lõi cho:
- **High Availability (HA):** Master chết? Promote Slave lên thay.
- **Scaling Reads:** Đẩy bớt load đọc báo cáo, analytics sang Slave.
- **Disaster Recovery:** Backup dữ liệu sang một data center khác.

Tuy nhiên, bản chất của nó là **asynchronous** (bất đồng bộ). Nghĩa là luôn có một độ trễ (replication lag) nhất định. Không ai dám cam kết Slave sẽ giống hệt Master tại mọi thời điểm cả.

Dưới đây là 3 chủ đề kỹ thuật quan trọng mà mình thấy anh em hay bỏ qua hoặc hiểu chưa sâu.

## 1. Cuộc chiến giữa các Replication Formats: Row vs Statement

Khi cấu hình `binlog_format`, chúng ta thường thấy 3 lựa chọn: `STATEMENT`, `ROW`, và `MIXED`. Chọn cái nào không phải là chuyện thích là nhích đâu nhé.

### Statement-Based
Đây là kiểu cổ điển. Master chạy query gì, nó ghi lại y chang query đó vào binlog. Slave nhận về và chạy lại.
- **Ưu điểm:** Log rất gọn nhẹ. Bạn update 1000 dòng thì log cũng chỉ là 1 câu SQL vài chục bytes.
- **Nhược điểm chết người:** **Nondeterministic**. Ví dụ bạn chạy `DELETE FROM users LIMIT 10` mà không có `ORDER BY`, thì Master có thể xóa 10 ông A, còn Slave lại xóa 10 ông B. Kết quả là dữ liệu lệch nhau vĩnh viễn (data drift) mà không hề hay biết. Rất nguy hiểm!

### Row-Based (Khuyên dùng)
Kiểu này ghi lại chính xác dòng nào đã bị thay đổi và giá trị mới là gì.
- **Ưu điểm:** An toàn tuyệt đối. Mọi thay đổi là tường minh (deterministic). Đây là chuẩn mực cho các hệ thống hiện đại.
- **Nhược điểm:** Log có thể phình to rất nhanh. Update 1 triệu dòng thì binlog sẽ ghi lại 1 triệu events thay đổi. Tuy nhiên, với băng thông mạng và dung lượng ổ cứng hiện nay thì đây là sự đánh đổi xứng đáng.

### Mixed
Cố gắng kết hợp cả hai. Mặc định là Statement, khi gặp query "nguy hiểm" thì chuyển sang Row. Nhưng thực tế thì nó khá "lằng nhằng" và khó debug.

**Lời khuyên:** Hãy dùng **Row-based** (`binlog_format=ROW`). Đừng tiếc chút dung lượng ổ cứng để đổi lấy sự an toàn của dữ liệu.

## 2. GTID: Cứu tinh cho vận hành (Ops)

Trước MySQL 5.6, anh em làm ops khổ sở vì phải nhớ `binlog file` và `position` (kiểu `mysql-bin.000001`, pos `12345`). Master chết, muốn dựng Slave mới lên thay? Phải mò mẫm xem nó chạy đến vị trí nào rồi, sai một ly đi một dặm.

**GTID (Global Transaction Identifier)** sinh ra để dẹp bỏ nỗi đau này. Mỗi transaction được gán một ID duy nhất (dạng `UUID:TransactionId`).
- Khi bật GTID, Slave không cần quan tâm file binlog tên là gì. Nó chỉ cần bảo Master: "Tao đã chạy đến GTID số X rồi, gửi cho tao mấy cái sau đó đi".
- Việc failover trở nên cực kỳ đơn giản (auto-positioning). Master chết, trỏ Slave A sang Slave B làm nguồn mới chỉ cần một câu lệnh, server tự biết phải sync từ đâu.

```sql
-- Cấu hình đơn giản để bật Auto Position (sau khi đã enable GTID mode)
CHANGE REPLICATION SOURCE TO SOURCE_AUTO_POSITION = 1;
```

Nếu anh em đang build hệ thống mới, **BẮT BUỘC** nên dùng GTID. Đừng quay lại thời kỳ đồ đá nữa.

## 3. Tăng tốc với Multithreaded Replication (MTS)

Một vấn đề kinh điển: Master thì mạnh, CPU 64 cores, write ầm ầm (multithreaded). Nhưng Slave thì chỉ có... **1 luồng SQL thread** để replay log. Kết quả là Slave chạy không kịp, replication lag ngày càng tăng (vài tiếng đồng hồ là bình thường).

Để giải quyết, MySQL giới thiệu **Multithreaded Replication (MTS)**.
Có 2 chế độ chính:
1.  **DATABASE:** Các thread khác nhau sẽ xử lý các DB khác nhau. Ngon nếu bạn chạy mô hình multi-tenant (mỗi khách hàng 1 DB). Nhưng nếu tất cả dồn vào 1 DB chính thì... vô dụng.
2.  **LOGICAL_CLOCK (Chân ái):** Đây mới là thứ chúng ta cần. Nó cho phép Slave chạy song song các transaction miễn là chúng thuộc cùng một "Binary Log Group Commit" trên Master. Hiểu nôm na là: nếu trên Master 2 transaction chạy song song được, thì trên Slave cũng thế.

**Config tham khảo:**
```ini
slave_parallel_workers = 4  # Hoặc 8, tùy số core
slave_parallel_type = LOGICAL_CLOCK
slave_preserve_commit_order = 1 # Quan trọng để đảm bảo thứ tự commit
```

## Các sai lầm thường gặp (Common Pitfalls)

### Tin sái cổ `Seconds_behind_source`
Mọi người hay dùng lệnh `SHOW REPLICA STATUS` và nhìn `Seconds_behind_source` = 0 rồi thở phào là hệ thống ngon.
Thực tế: Số này cực kỳ ảo ma. Nếu Slave mất kết nối mạng hoặc thread SQL bị treo, nó có thể báo NULL hoặc 0. Hoặc khi chạy một long-running transaction (ví dụ 1 tiếng), thì trong suốt 1 tiếng đó lag = 0, đến khi commit xong nó mới vọt lên 3600s. Đừng tin nó tuyệt đối.

### Replication Filters sai chỗ
Dùng `binlog_do_db` (trên Master) là một cái bẫy.
Ví dụ bạn config `binlog_do_db = sale`.
Bạn chạy:
```sql
USE admin;
UPDATE sale.orders SET status = 1;
```
Bùm! Câu lệnh này **KHÔNG** được ghi vào binlog vì DB hiện tại đang là `admin` chứ không phải `sale`. Dữ liệu trên Slave sẽ bị thiếu.
Luôn ưu tiên filter trên Slave (`replicate_do_db`) thay vì trên Master.

## Kinh nghiệm thực tế / Lesson Learned

### Crash-Safe là sống còn
Đã làm replication thì phải tính đến đường Master sập nguồn (mất điện, crash OS). Để không bị mất dữ liệu hoặc corrupt relay log, bộ settings này là must-have:
- `sync_binlog = 1`: Ép ghi binlog ra đĩa sau mỗi transaction.
- `innodb_flush_log_at_trx_commit = 1`: Chuẩn ACID cho InnoDB.
- `relay_log_info_repository = TABLE`: Lưu trạng thái replication vào bảng (transactional) thay vì file.
- `relay_log_recovery = ON`: Khi Slave khởi động lại sau crash, nó sẽ bỏ qua các relay log hỏng và xin lại từ Master.

### Delayed Replication - Nút "Undo" của DB
Mình thường setup thêm một con Slave đặc biệt, cấu hình `SOURCE_DELAY = 3600` (chậm 1 tiếng).
Để làm gì? Để khi lỡ tay `DROP TABLE` hay update sai cả triệu dòng trên Master, mình vẫn còn 1 tiếng để chạy vào con Slave này lấy lại dữ liệu trước khi thảm họa lan tới nó. Backup định kỳ là tốt, nhưng restore mất cả ngày. Delayed Replica restore chỉ mất vài phút.

## Kết luận

Replication trong MySQL là một con dao hai lưỡi. Dùng đúng thì hệ thống scale ngon lành, HA xịn sò. Dùng sai thì data lệch, debug muốn trầm cảm. Tư duy đúng ở đây là: **An toàn dữ liệu (Consistency) > Tốc độ**. Hãy ưu tiên Row-based, GTID và các settings crash-safe trước khi nghĩ đến việc tune performance.

Anh em nào muốn tìm hiểu sâu hơn thì có thể đọc tiếp về chủ đề **Orchestrator** (tool quản lý topology) hoặc **Group Replication** (tương lai của MySQL HA).

Happy coding! (hẹ hẹ)

**Nguồn:**
- High Performance MySQL, 4th Edition – Chapter 9: Replication
- Official MySQL Documentation