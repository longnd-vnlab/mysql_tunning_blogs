# Chuyện Compliance: Không chỉ là giấy tờ, mà là sự sống còn của hệ thống (Chapter 13 Breakdown)

## Mở đầu – Tản mạn dev

Chắc hẳn anh em làm backend hay infra đều từng trải qua cảm giác "lạnh sống lưng" khi nhận email từ bộ phận Security hoặc team Audit kiểu: "Tại sao user DB này quyền root?", "Ai đã DROP bảng user đêm qua?", hay căng hơn là "Chứng minh cho tôi thấy hệ thống của anh đạt chuẩn PCI DSS/SOC 2 để ký hợp đồng với khách hàng Enterprise".

Thú thật, ngày xưa mình cũng nghĩ Compliance (tuân thủ) là việc của mấy ông "giấy tờ", dev cứ code chạy ngon là được. Nhưng rồi đến lúc hệ thống scale lên, khách hàng đòi hỏi khắt khe về bảo mật dữ liệu, thì mớ "nợ kỹ thuật" về quản lý access, log audit nó quay lại "bón hành" cực mạnh. (haizz)

Bài viết này mình chia sẻ lại những điểm cốt lõi nhất từ **Chapter 13: Compliance with MySQL** trong cuốn *High Performance MySQL, 4th Edition*, kết hợp với chút kinh nghiệm đau thương của bản thân. Hy vọng giúp anh em xây dựng mindset đúng đắn về compliance ngay từ đầu, tránh cảnh "mất bò mới lo làm chuồng".

## Tổng quan

Trong môi trường Enterprise, Compliance không chỉ là việc tích vào checklist cho xong. Nó là nền tảng để đảm bảo tính toàn vẹn dữ liệu (Data Integrity) và khả năng phục hồi (Resilience). Các chuẩn như SOC 2, PCI DSS, HIPAA hay GDPR không sinh ra để làm khó dev, mà để ép chúng ta phải có quy trình minh bạch.

Concept cốt lõi ở đây xoay quanh ba trụ cột:
1.  **Kiểm soát truy cập (Access Control):** Ai được vào đâu, làm gì?
2.  **Theo dõi thay đổi (Change Tracking):** Ai đã sửa gì, vào lúc nào?
3.  **Khả năng phục hồi (Recoverability):** Backup có chạy không, restore có lên không?

Nếu anh em nghĩ chỉ cần `GRANT ALL PRIVILEGES` cho tiện thì đọc tiếp nhé, phần sau sẽ khá thú vị đấy.

## Quản lý Secrets và Xoay vòng Credential không Downtime

Một trong những bài toán "nhức đầu" nhất khi làm compliance là **Credential Rotation** (xoay vòng mật khẩu). Theo chuẩn (ví dụ PCI DSS), anh em phải đổi password DB định kỳ (ví dụ 90 ngày).

Cách làm truyền thống (và sai lầm) là:
1.  Đổi pass trong DB.
2.  Hệ thống sập (downtime) do app chưa cập nhật config.
3.  Deploy lại app với pass mới.
4.  App chạy lại.

Cách này với hệ thống lớn là không chấp nhận được. Trong sách có đề cập một kỹ thuật cực hay từ MySQL 8.0.14 trở đi: **Dual Password Support**. Tính năng này cho phép một user giữ 2 password cùng lúc (một cũ, một mới).

Quy trình "ngon lành" sẽ như sau:
1.  Tạo password mới cho user (giữ lại pass cũ làm secondary).
2.  App vẫn chạy bình thường với pass cũ.
3.  Update config app sang pass mới và redeploy dần dần.
4.  Khi chắc chắn 100% app đã dùng pass mới, ta discard pass cũ đi.

Anh em thấy không, zero downtime, không ai phải thức đêm canh deploy chỉ để đổi pass cả. Đây là một ví dụ điển hình của việc hiểu tính năng DB giúp mình nhàn hạ hơn bao nhiêu.

## Audit Logging: Đừng dùng Triggers!

Khi sếp yêu cầu: "Ghi log lại tất cả ai sửa bảng `financial_reports`", phản xạ đầu tiên của nhiều anh em là: "À, viết cái Trigger `AFTER UPDATE` là xong".

Đừng nhé! (sad)

Tác giả cuốn sách phân tích rất kỹ tại sao Trigger là một lựa chọn tồi cho Audit Log:
1.  **Hiệu năng:** Trigger gây overhead cực lớn lên thao tác Write. Lúc bình thường không sao, lúc cao điểm traffic đổ về, DB treo cứng vì phải gánh thêm logic log lằng nhằng.
2.  **Khó quản lý:** Trigger nằm ẩn trong DB, code nằm một nơi, logic nằm một nẻo. Khi debug hay migrate dữ liệu rất dễ quên hoặc gây side-effect không mong muốn.
3.  **Giới hạn:** Trigger chỉ bắt được Write (Insert/Update/Delete), còn Read (Select) thì chịu chết. Mà compliance nhiều khi yêu cầu log cả việc "Ai đã xem dữ liệu nhạy cảm này?".

Giải pháp "chuẩn bài" là dùng **Audit Plugin** (như MySQL Enterprise Audit hoặc Percona Audit Log Plugin). Nó hoạt động ở tầng Server, hook trực tiếp vào quá trình thực thi query, nhẹ nhàng hơn và bắt được đủ mọi loại event.

Mẹo nhỏ: Đừng log ra file text trên server DB rồi để đó. Hãy cấu hình đẩy log qua `rsyslog` về hệ thống tập trung (ELK, Splunk) để team Security dễ dàng query và alert.

## Vệ sinh User định kỳ (User Access Hygiene)

Có một lỗi sơ đẳng mà rất nhiều team gặp phải: User của nhân viên đã nghỉ việc 6 tháng vẫn còn quyền access vào Production DB. Hoặc một service đã decommission từ đời nào nhưng user DB vẫn còn active. Đây là lỗ hổng bảo mật cực lớn ("security liability").

Làm sao để biết user nào là "rác"?
Trong MySQL, chúng ta có thể tận dụng `Performance Schema`. Cụ thể là bảng `performance_schema.users`. Bảng này track lịch sử connect của các user.

Anh em có thể định kỳ chạy script kiểm tra: Nếu user X đã được tạo > 6 tháng mà không xuất hiện trong bảng history connect, hoặc lần cuối connect đã quá lâu -> khả năng cao là user thừa -> **DROP ngay và luôn**.

Tư duy ở đây là **Least Privilege** (Quyền tối thiểu). Chỉ cấp quyền vừa đủ dùng, và thu hồi ngay khi không cần thiết.

## Các sai lầm thường gặp (Common Pitfalls)

1.  **Dùng chung User:** Cả team dev dùng chung một user `root` hoặc `admin`. Khi có sự cố, không ai biết do ông nào gây ra (dù log có ghi lại thì cũng chỉ thấy `root` làm).
2.  **Lưu Credentials trong Code:** Hardcode password trong git repo. Cái này thì... ối dời ơi luôn. Hãy dùng các giải pháp Secret Management (Vault, AWS Secrets Manager) để inject vào lúc runtime.
3.  **Backup nhưng không Test Restore:** Có setup backup chạy cronjob hàng ngày rất đều đặn. Nhưng đến khi crash thật, lôi file backup ra restore thì lỗi file hoặc sai version. (toang).
4.  **Log quá nhiều:** Bật Audit Log chế độ "ghi tất cả". Kết quả là ổ cứng đầy sau 2 tiếng và DB crash vì I/O quá tải. Hãy filter kỹ những gì cần log (ví dụ chỉ log lệnh DDL, hoặc truy cập bảng nhạy cảm).

## Kinh nghiệm thực tế / Lesson Learned

Hồi mình làm dự án Fintech, team có một lần bị auditor "soi" vụ schema change. Họ hỏi: "Làm sao anh chứng minh được thay đổi cấu trúc bảng `transactions` hôm qua là đã được phê duyệt và test?".

Lúc đó team mình hay SSH vào server rồi gõ lệnh `ALTER TABLE` tay (thói quen xấu khó bỏ). Kết quả là ú ớ không đưa ra được bằng chứng.

Sau vụ đó, bọn mình phải chuyển sang mô hình **Database Migration as Code** (dùng Flyway/Liquibase).
- Mọi thay đổi schema là một file SQL committed vào Git.
- Phải qua Pull Request review.
- CI/CD tự động chạy migration.
- Không ai được phép SSH vào DB production để sửa schema nữa.

Kết quả là lần audit sau, chỉ cần show lịch sử Git Commit và Pipeline log ra là auditor gật đầu cái rụp. Nhẹ cả người.

Một bài học nữa là về **Data Sovereignty** (Chủ quyền dữ liệu). Có những khách hàng yêu cầu dữ liệu của họ không được ra khỏi lãnh thổ quốc gia họ. Lúc này kiến trúc DB phải tính đến việc Sharding theo location hoặc tách cụm DB riêng biệt, chứ không đơn thuần là replication toàn cầu được nữa.

## Kết luận

Compliance với MySQL không phải là mua một tool về cài là xong. Nó là sự kết hợp giữa:
1.  **Tính năng kỹ thuật:** Dual password, Audit Plugin, Performance Schema.
2.  **Quy trình:** Review code, CI/CD cho DB, Rotation policy.
3.  **Mindset:** "Mọi thay đổi đều phải được theo dõi và giải trình được".

Anh em đừng đợi đến lúc bị phạt hay bị hack mới lo làm. Hãy bắt đầu từ những việc nhỏ như rà soát lại list user, bỏ thói quen dùng chung account, và script hóa quy trình backup/restore.

Nếu anh em muốn tìm hiểu sâu hơn, có thể đào thêm về topic: *Hashicorp Vault integration with MySQL* để quản lý dynamic secrets, bao ngầu và bao an toàn.

**Nguồn tham khảo:**
- High Performance MySQL, 4th Edition – Chapter 13: Compliance with MySQL
- Official MySQL Documentation