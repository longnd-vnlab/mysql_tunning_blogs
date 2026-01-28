# [MySQL & SRE] Monitoring kiểu Reliability Engineering: Đừng chỉ nhìn vào CPU & RAM!

## Mở đầu – Tản mạn dev
Có bao giờ anh em gặp cảnh hệ thống đang chạy ngon lành, dashboard xanh lét, CPU mới load 20%, RAM còn đầy cả rổ, thế mà khách hàng vẫn réo ầm ĩ vì "web quay đều, quay đều" không? (sad)

Mình cá là nhiều anh em dev/sysadmin từng rơi vào thế khó xử này: Monitor báo server khỏe re, nhưng trải nghiệm người dùng thì tệ hại. Đó là lúc chúng ta nhận ra cách monitor truyền thống theo kiểu "server sống hay chết" đã lỗi thời. Hôm nay, mình sẽ chia sẻ với mọi người góc nhìn từ Chapter 2 cuốn "High Performance MySQL, 4th Edition" về việc áp dụng tư duy SRE vào giám sát database. Bài này sẽ giúp anh em chuyển từ việc monitor cho máy xem sang monitor để người dùng sướng.

## Tổng quan
Trong kỷ nguyên của Site Reliability Engineering (SRE), vai trò của việc giám sát MySQL không chỉ dừng lại ở việc tối ưu từng câu query hay cấu hình server (dù cái đó vẫn cần). Quan trọng hơn, chúng ta cần trả lời được câu hỏi: **"Khách hàng có đang happy không?"**.

Monitor hiện đại tập trung vào **Outcomes** (kết quả trải nghiệm) thay vì **Outputs** (thông số kỹ thuật). Thay vì chỉ chăm chăm nhìn vào `iostat` hay `top`, chúng ta cần xây dựng bộ chỉ số dựa trên **SLI (Service Level Indicators)** và **SLO (Service Level Objectives)**.

Dưới đây là 3 chủ đề cốt lõi mình rút ra được để anh em "nâng trình" monitoring của mình.

## 1. Định nghĩa SLI/SLO: Đừng mơ về con số 100%
Nhiều sếp hay yêu cầu: "Hệ thống phải uptime 100% nhé!". Thực tế là... mơ đi cưng (hẹ hẹ). Trong thế giới SRE, failure là điều tất yếu.

Thay vì hứa hẹn viển vông, anh em cần xác định:
*   **SLI (Chỉ số mức dịch vụ):** Là thước đo sự hạnh phúc của user. Ví dụ: Tỷ lệ request thành công, độ trễ phản hồi.
*   **SLO (Mục tiêu mức dịch vụ):** Là ngưỡng chấp nhận được. Ví dụ: 99.5% request hoàn thành dưới 200ms trong 1 tháng.

**Tại sao lại là 99.5% mà không phải 100%?**
Vì để đạt được vài số 9 lẻ đằng sau (như 99.999%), chi phí kỹ thuật và nhân sự sẽ tăng theo cấp số nhân. Anh em cần "Error Budget" (ngân sách lỗi) để team còn dám thử nghiệm tính năng mới. Nếu cứ bắt cam kết 100%, dev sẽ sợ "đụng đâu hỏng đó" và chả dám deploy gì sất.

## 2. Giám sát độ trễ (Latency) & Lỗi (Errors): Client là chân ái
Một sai lầm kinh điển là chỉ đo latency ngay trên DB server.
*   **Server bảo:** "Tao xử lý query này mất có 1ms à, nhanh vãi chưởng".
*   **Client (App) bảo:** "Nhanh cái rổ, tao mất 2 giây mới nhận được phản hồi do mạng lag, do queue, do driver lằng nhằng".

**Kinh nghiệm xương máu:**
Hãy đo latency từ phía **Client/Application**. Đó mới là con số thật mà user phải chịu đựng. Anh em nên dùng các công cụ APM hoặc custom metrics từ code để đo thời gian từ lúc gửi query đến lúc nhận kết quả trọn vẹn.

Về **Errors**, không phải lỗi nào cũng đáng báo động.
*   *Lỗi "nhiễu":* Deadlock, timeout thỉnh thoảng xảy ra rồi retry là hết -> Kệ nó.
*   *Lỗi "tín hiệu":* `Connection refused`, `Read-only mode` (khi đang write), hoặc số lượng `Aborted connections` tăng đột biến -> Báo động ngay! Đây là dấu hiệu hệ thống sắp "bùm".

## 3. Proactive Monitoring: Bắt bệnh trước khi nhập viện
Đợi đến lúc disk full 100% hay CPU 100% mới alert thì gọi là... "vuốt đuôi". Anh em cần theo dõi các **Leading Indicators** (chỉ báo sớm).

**Hai chỉ số "thần thánh" cần soi kỹ:**
1.  **Threads_running:** Đây là số lượng query đang thực sự chạy đồng thời. Nếu con số này tăng dựng đứng và không giảm, nghĩa là DB đang bị kẹt (CPU lockup hoặc disk I/O quá tải). Server sắp treo rồi đấy!
2.  **Connection Growth:**
    *   Monitor tỷ lệ `Threads_connected / Max_connections`.
    *   Đừng set `Max_connections` quá cao vô tội vạ. Nếu app mở connection vô tội vạ mà không reuse (connection leak), DB sẽ sập nguồn vì hết RAM trước khi hết slot connect.

**Về Disk I/O:**
Đừng chỉ alert khi ổ cứng đầy 90%. Hãy monitor **tốc độ tăng trưởng** (growth rate). Ví dụ: Ổ cứng còn 20% nhưng tốc độ ghi dữ liệu đang là 10GB/giờ -> 2 tiếng nữa là "tạch". Alert ngay!

## Các sai lầm thường gặp (Common Pitfalls)
*   **Trung bình cộng (Averages) là cú lừa:** Đừng bao giờ nhìn vào Average Latency. Một request mất 10s có thể bị chìm nghỉm trong 1000 request mất 1ms. Hãy dùng **Percentiles (P95, P99)**. P99 bảo đảm rằng 99% user đều sướng, chứ không phải "trung bình user thấy sướng".
*   **Alert mỏi mồm (Alert Fatigue):** Cấu hình alert cho mọi thứ. Kết quả là điện thoại rung cả ngày, anh em tắt luôn thông báo. Chỉ alert những gì **Actionable** (cần hành động ngay) và ảnh hưởng trực tiếp đến user.
*   **Sợ "Test in Production":** Nghe thì ghê nhưng SRE khuyến khích test cẩn thận trên production (Canary release, Feature flag). Môi trường Staging không bao giờ giống thật 100% đâu.

## Kinh nghiệm thực tế / Lesson Learned
Hồi xưa mình làm dự án E-commerce, cứ đến Black Friday là hệ thống "rụng". Lý do là anh em không tính đến **Business Cadence** (nhịp sinh học của doanh nghiệp).
*   Lúc bình thường, Replication Lag vài giây chả sao.
*   Nhưng lúc sale, Lag vài giây nghĩa là khách vừa thanh toán xong, F5 lại thấy đơn hàng "chưa thanh toán" (do đọc từ Slave chưa kịp sync). Khách hoảng loạn, support quá tải.

**Bài học:** Replication Lag không chỉ là con số kỹ thuật, nó là trải nghiệm khách hàng. Nếu lag cao, phải có cơ chế fallback (đọc từ Master) hoặc thông báo khéo léo cho user, chứ đừng để họ tưởng mất tiền oan.

Một case nữa là **Connection Storm**. App server thấy chậm -> mở thêm connection -> DB càng chậm -> App lại mở thêm connection. Cái vòng luẩn quẩn này sẽ giết chết DB trong 3 nốt nhạc. Giải pháp là phải có Connection Pool xịn và set timeout hợp lý ở phía App.

## Kết luận
Monitor MySQL theo phong cách SRE không phải là vứt bỏ các công cụ cũ, mà là thay đổi tư duy: **Lấy user làm trung tâm**.
*   Đừng bị ám ảnh bởi uptime 100%.
*   Hãy soi P95, P99 thay vì Average.
*   Monitor từ phía App để thấy nỗi đau của user.

Chủ đề tiếp theo anh em nên tìm hiểu là **Performance Schema** (Chapter 3 trong sách), cái này giúp soi sâu vào nội tạng MySQL cực hay, tìm ra vì sao query chậm mà không cần đoán mò. Chúc anh em monitor ngon lành, ngủ ngon không bị pager réo! (haizz)

**Nguồn:**
*   High Performance MySQL, 4th Edition – Chapter 2: Monitoring in a Reliability Engineering World[1]
*   Official MySQL Documentation