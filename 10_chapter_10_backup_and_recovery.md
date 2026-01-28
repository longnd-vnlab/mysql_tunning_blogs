# [MySQL Backup & Recovery] Đừng để "mất bò mới lo làm chuồng" - Chuyện backup không chỉ là copy file

## Mở đầu – Tản mạn dev
Có bao giờ anh em rơi vào cảnh toát mồ hôi hột lúc 3 giờ sáng vì lỡ tay `DROP TABLE` nhầm trên production, hay ổ cứng con DB server chính lăn đùng ra chết chưa? Mình thì rồi, và cảm giác đó thốn không tả nổi. Lúc đó mới thấy cái backup nó quý hơn vàng. Nhưng mà backup thế nào cho chuẩn, restore làm sao cho nhanh thì lại là câu chuyện dài hơi mà nhiều khi anh em dev mình hay bỏ qua, cứ nghĩ "có script chạy cronjob hàng đêm là ngon rồi".

Hôm nay mình sẽ chia sẻ với mọi người về Backup và Recovery trong MySQL, dựa trên những gì mình ngâm cứu được từ sách "High Performance MySQL" và kinh nghiệm chinh chiến thực tế. Bài viết này sẽ giúp anh em hiểu bản chất các loại backup, cách dùng tool xịn xò để không phải lock DB, và quan trọng nhất là tư duy để ngủ ngon mỗi tối.

## Tổng quan
Trong thế giới database, backup không chỉ đơn thuần là copy dữ liệu ra một chỗ khác để dành. Nó là tấm khiên cuối cùng bảo vệ hệ thống trước thảm họa (disaster recovery), là nguồn để audit khi có biến, và đôi khi là để dựng môi trường staging test bug.[1]

Hai khái niệm cốt lõi anh em cần nắm vững trước khi đụng vào bất kỳ tool nào là **RPO (Recovery Point Objective)** và **RTO (Recovery Time Objective)**. Hiểu nôm na:
- **RPO**: Anh em chấp nhận mất tối đa bao nhiêu dữ liệu? (5 phút, 1 tiếng, hay không được mất transaction nào?).
- **RTO**: Hệ thống được phép sập trong bao lâu để restore? (Nếu sếp bảo "phải lên lại ngay trong 5 phút" mà data 1TB thì căng đấy).[1]

Xác định được hai cái này thì mới chọn được chiến lược backup phù hợp, chứ không phải cứ `mysqldump` là xong chuyện đâu nhé.

## Chủ đề 1: Logical Backup vs. Raw Backup - Cuộc chiến không hồi kết
Khi bàn về backup MySQL, câu hỏi đầu tiên luôn là: "Nên dùng `mysqldump` hay copy file?"

**Logical Backup (Backup logic)**
Đây là cách dùng `mysqldump` hoặc `mydumper` để xuất dữ liệu ra dạng câu lệnh SQL (`CREATE TABLE`, `INSERT`...).
- **Ưu điểm:** File text nên dễ nhìn, dễ sửa (nếu cần), convert sang server khác version hoặc khác OS dễ dàng. Nó cũng giúp loại bỏ việc corruption ở tầng vật lý.[1]
- **Nhược điểm:** Chậm. Rất chậm. Cả lúc backup lẫn lúc restore. Khi restore, server phải chạy lại hàng triệu câu lệnh SQL, build lại index từ đầu. Anh em thử tưởng tượng restore 100GB data bằng cách chạy `INSERT` từng dòng thì chờ đến "mùa quýt".[1]

**Raw Backup (Backup vật lý)**
Là copy trực tiếp các file dữ liệu của MySQL (`.ibd`, `ibdata1`...) ra chỗ khác.
- **Ưu điểm:** Tốc độ bàn thờ. Vì nó là copy block ổ cứng nên nhanh hơn nhiều, đặc biệt là lúc restore. Chỉ cần copy file về, start MySQL lên là xong (gần như vậy).[1]
- **Nhược điểm:** File thường to hơn logical backup. Kén chọn OS và version MySQL hơn. Và quan trọng là phải đảm bảo tính nhất quán (consistency) lúc copy, chứ đang copy mà có thằng ghi vào file thì file đó coi như bỏ.[1]

**Lời khuyên:** Database nhỏ (< 10GB) thì `mysqldump` cho tiện. Nhưng khi data đã lớn, anh em buộc phải chuyển sang Raw Backup nếu không muốn RTO kéo dài cả ngày.

## Chủ đề 2: Percona XtraBackup - Vũ khí hạng nặng cho Hot Backup
Nếu chọn Raw Backup mà anh em định `cp` hay `tar` thư mục data lúc MySQL đang chạy thì... toang nhé. Dữ liệu sẽ không nhất quán và corrupt ngay. Để làm Raw Backup mà không cần stop server (Hot Backup), `Percona XtraBackup` là tiêu chuẩn ngành rồi.

Cơ chế của nó cực hay:
1.  Nó sẽ copy các data file (`.ibd`) trong background.
2.  Trong quá trình copy, nếu có transaction mới ghi vào DB, MySQL sẽ ghi vào Redo Log. XtraBackup sẽ có một thread chạy song song để "hóng" (tail) cái Redo Log này và ghi lại.[1]
3.  Sau khi copy xong file data, nó sẽ có một mớ Redo Log chưa được apply vào file backup.

Lúc này file backup đang ở trạng thái "bừa bộn" (inconsistent). Bước quan trọng tiếp theo là **Prepare**:
```bash
xtrabackup --prepare --target-dir=/backup/path
```
Bước này sẽ replay các Redo Log vào data file, giống như cơ chế Crash Recovery của InnoDB vậy. Sau bước này, file backup mới thực sự dùng được.[1]

Cái hay của XtraBackup là nó không lock table (với InnoDB), nên production vẫn chạy phà phà, anh em không lo bị block user.

## Chủ đề 3: Point-in-Time Recovery (PITR) - Cỗ máy thời gian
Giả sử anh em backup full lúc 12h đêm. Đến 10h sáng hôm sau, một dev lỡ tay xóa nhầm bảng `users`. Nếu restore bản backup đêm qua, anh em mất sạch dữ liệu từ 12h đêm đến 10h sáng. Toang tập 2.

Đây là lúc **Binary Log (binlog)** tỏa sáng. Binlog ghi lại toàn bộ các thay đổi dữ liệu theo thời gian thực.
Chiến thuật sẽ là:
1.  Restore bản full backup lúc 12h đêm.
2.  Dùng công cụ `mysqlbinlog` để replay lại toàn bộ sự kiện từ 12h đêm đến 9:59 (trước khi lệnh xóa bảng diễn ra).[1]

Để làm được việc này, anh em BẮT BUỘC phải bật `log_bin` trong config và backup cả binlog thường xuyên (ví dụ sync ra S3 mỗi 5 phút). Đây chính là chìa khóa để đạt được RPO thấp nhất có thể.

## Các sai lầm thường gặp (Common Pitfalls)

**1. "Replication là Backup" - Sai lầm chết người**
Rất nhiều anh em nghĩ có Slave (Replica) rồi thì cần gì backup. Sai hoàn toàn nhé. Nếu ai đó chạy lệnh `DROP DATABASE` trên Master, lệnh đó sẽ được replicate xuống Slave trong tích tắc. Kết quả là cả 2 nơi đều mất sạch dữ liệu. Replica chỉ giúp High Availability, không giúp Disaster Recovery.[1]

**2. "Snapshot ổ cứng là Backup" - Chưa đủ**
Anh em dùng LVM hay EBS Snapshot của AWS thấy tiện quá, bấm cái bùm là xong. Nhưng coi chừng, snapshot (copy-on-write) chỉ lưu sự khác biệt. Nếu block dữ liệu gốc bị corrupt (bad sector), thì snapshot cũng trỏ vào cái block hỏng đó thôi. Snapshot tốt cho việc lấy nhanh một bản copy tại thời điểm, nhưng đừng tin tưởng nó 100% để lưu trữ lâu dài.[1]

**3. Backup xong... để đó**
Có file backup nhưng không bao giờ thử restore. Đến lúc cần dùng mới phát hiện file bị lỗi, hoặc thời gian restore mất 3 ngày trong khi sếp chỉ cho 3 tiếng. Đây gọi là "Schrödinger's backup" - backup vừa tồn tại vừa không tồn tại cho đến khi bạn thử restore nó.

## Kinh nghiệm thực tế / Lesson Learned
Hồi mình làm hệ thống cho một bên e-commerce, data khoảng 500GB. Ban đầu dùng `mysqldump` chạy mỗi đêm. Mọi thứ vẫn ổn cho đến khi cần dựng một con staging giống hệt production để test tính năng mới. Team hì hục restore cái file dump đó mất gần... 12 tiếng đồng hồ. Cả team ngồi nhìn màn hình console mà ngán ngẩm.

Sau vụ đó, mình chuyển sang dùng Percona XtraBackup và kết hợp streaming đẩy thẳng lên S3 (dùng `xbstream`). Thời gian backup giảm xuống còn 1 tiếng, và restore chỉ mất khoảng 2-3 tiếng (do tốn thời gian kéo từ S3 về).

Một kinh nghiệm xương máu nữa: Khi restore Raw Backup, nhớ kiểm tra kỹ **quyền sở hữu file (chown mysql:mysql)**. Copy file về mà quên set quyền, MySQL không đọc được file rồi crash log lỗi tùm lum, debug lòi mắt mới ra.[1]

## Kết luận
Backup và Recovery là việc tẻ nhạt nhưng sống còn. Tư duy đúng là: **Đừng chỉ backup cho có, hãy backup để có thể recover.**
- Data nhỏ thì cứ `mysqldump` cho nhàn.
- Data lớn, cần RTO thấp thì Percona XtraBackup là chân ái.
- Luôn bật Binary Log để cứu vớt dữ liệu đến từng giây.
- Đừng tin vào Replica, hãy tin vào file backup đã được test restore thành công.

Anh em nào muốn tìm hiểu sâu hơn thì có thể đọc kỹ Chapter 10 trong cuốn "High Performance MySQL, 4th Edition", phần nói về *Lock-free InnoDB backups* và *Incremental backups* rất hay ho để tối ưu thêm. Chúc anh em vận hành hệ thống ngon lành, ngủ ngon mỗi tối![1]

**Nguồn:**
- High Performance MySQL, 4th Edition – Chapter 10: Backup and Recovery
- Official MySQL Documentation