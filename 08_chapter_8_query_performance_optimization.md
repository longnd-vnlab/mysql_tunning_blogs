# Query Performance Optimization: Khi Index Là Chưa Đủ

## Mở đầu – Tản mạn dev

Chắc anh em làm backend không lạ gì cảnh tượng này: Hệ thống đang chạy ngon, tự nhiên một ngày đẹp trời cái dashboard load mãi không ra, khách hàng thì réo ầm ĩ. Vào check log thì thấy Slow Query Log dài như sớ Táo Quân. Mở code ra xem thì... ôi thôi, một câu SQL dài dằng dặc, JOIN 5-7 bảng, lại còn `GROUP BY`, `ORDER BY` đủ cả.

Anh em vội vàng add thêm cái index, cầu mong phép màu xảy ra. May thì nó nhanh lên chút, xui thì vẫn rùa bò. Lúc này mới nhận ra: Index không phải là thuốc tiên. Vấn đề đôi khi nằm ngay ở cách chúng ta viết query và tư duy lấy dữ liệu. Bài viết này mình sẽ chia sẻ về tư duy tối ưu query (Query Performance Optimization) dựa trên Chapter 8 của cuốn "High Performance MySQL, 4th Edition" - một cuốn sách gối đầu giường mà mình rất tâm đắc.

## Tổng quan

Trong MySQL, tối ưu query không chỉ đơn giản là "viết sao cho chạy". Nó là cuộc chơi của việc quản lý **Response Time** (thời gian phản hồi). Về bản chất, Response Time = Service Time (thời gian thực thi) + Queue Time (thời gian chờ).

Chúng ta thường chỉ tập trung vào Service Time mà quên mất rằng nếu query viết dở, nó sẽ lock resources, làm tăng Queue Time của các request khác, kéo sập cả hệ thống. Core concept ở đây là: **Query nhanh là query xử lý ít dữ liệu nhất có thể**.

Dưới đây là 3 kỹ thuật cốt lõi mình rút ra được để anh em áp dụng ngay vào dự án.

## 1. Tối ưu Data Access: Đừng tham lam

Nguyên nhân phổ biến nhất khiến query chậm là vì nó đang phải truy cập quá nhiều dữ liệu. Nghe hiển nhiên đúng không? Nhưng check lại xem anh em có dính 2 lỗi này không nhé:

**Lấy thừa cột (Columns):**
Cái thói quen `SELECT *` là kẻ thù số một. Khi anh em select `*`, database engine không thể dùng Covering Index (kỹ thuật chỉ đọc dữ liệu từ index mà không cần chọc vào bảng chính). Chưa kể nó còn kéo theo cả đống dữ liệu rác qua network, tăng I/O và CPU.
*   *Lời khuyên:* Cần cột nào, select đúng cột đó. Thậm chí nếu anh em chỉ cần check sự tồn tại, `SELECT 1` còn ngon hơn.

**Quét thừa dòng (Rows):**
Đây là chỉ số quan trọng nhất: **Rows examined vs. Rows returned**.
Lý tưởng nhất là tỉ lệ 1:1, tức là anh em quét 10 dòng và trả về đúng 10 dòng. Nhưng thực tế thường phũ phàng, có khi anh em quét 10.000 dòng chỉ để lọc ra 5 dòng kết quả (tỉ lệ 2000:1).
*   *Giải pháp:* Nếu tỉ lệ này quá cao, nghĩa là index đang không hiệu quả hoặc cấu trúc bảng có vấn đề. Hãy xem lại `WHERE` clause, đảm bảo nó filter dữ liệu ngay từ đầu chứ không phải đợi load lên hết rồi mới lọc.

## 2. Chiến thuật "Chia để trị" (Restructuring Queries)

Hồi mới làm dev, mình luôn nghĩ: "Gửi 1 query to chắc chắn nhanh hơn gửi 10 query nhỏ vì đỡ tốn công kết nối database". Tư duy này với MySQL hiện đại là **SAI**. Kết nối MySQL bây giờ rất rẻ và nhanh.

**Complex vs. Many Queries:**
Đôi khi, việc xé nhỏ một query phức tạp thành nhiều query đơn giản lại hiệu quả hơn. Ví dụ điển hình là **Join Decomposition** (Phân rã JOIN).
Thay vì JOIN 3 bảng `Tag`, `Product`, `Category` trong một query duy nhất, anh em có thể:
1.  Select `Tag` lấy list `product_id`.
2.  Select `Product` dựa trên list id vừa lấy.
3.  Select `Category` tương tự.

Nghe có vẻ "đi ngược sách giáo khoa" nhưng lợi ích của nó cực lớn: caching hiệu quả hơn (cache từng bảng đơn lẻ), giảm lock contention, và dễ dàng scale hoặc sharding sau này. Ứng dụng code PHP/Go/Java xử lý list ID trong memory nhanh hơn nhiều so với bắt DB gồng mình JOIN.

**Chopping up a query (Cắt nhỏ query):**
Điển hình là vụ xóa dữ liệu cũ. Anh em chạy `DELETE FROM logs WHERE date < '2023-01-01'` phát là DB treo luôn vì nó lock cả triệu dòng.
Cách làm chuẩn architect: Viết một script loop, mỗi lần xóa 10.000 dòng, sleep 1-2 giây rồi xóa tiếp. Vừa không lock bảng lâu, vừa không làm đầy Transaction Log.

## 3. "Ảo thuật" với GROUP BY và LIMIT

Đây là hai tác vụ "ngốn" tài nguyên kinh khủng nếu không để ý.

**Tối ưu COUNT():**
Có một huyền thoại anh em hay truyền tai nhau: "Dùng `COUNT(id)` nhanh hơn `COUNT(*)`". Sai bét nhé.
Trong MySQL, `COUNT(*)` được tối ưu đặc biệt để đếm số dòng (rows). Còn `COUNT(col)` sẽ phải check thêm điều kiện column đó có NULL hay không. Nên trừ khi anh em muốn đếm giá trị khác NULL, cứ `COUNT(*)` mà phang.
*Note:* Với InnoDB, `COUNT(*)` không hề "instant" như MyISAM ngày xưa đâu, nó vẫn phải quét index đấy. Nếu cần đếm xấp xỉ cho nhanh (ví dụ hiển thị số page), hãy dùng `EXPLAIN` hoặc meta tables.

**Tối ưu Pagination (Phân trang):**
Vấn đề kinh điển: `LIMIT 10000, 20`.
Để lấy 20 dòng ở trang thứ 500, MySQL thực chất phải quét 10.020 dòng, rồi vứt bỏ 10.000 dòng đầu. Càng về trang sau càng chậm (offset càng lớn càng chết).
*   *Cách fix (Deferred Join):*
    Thay vì:
    ```sql
    SELECT * FROM users ORDER BY created_at LIMIT 10000, 20;
    ```
    Hãy dùng subquery để lấy ID trước (tận dụng covering index):
    ```sql
    SELECT * FROM users u
    INNER JOIN (
        SELECT id FROM users ORDER BY created_at LIMIT 10000, 20
    ) AS lim ON u.id = lim.id;
    ```
    Nó chỉ phải lôi data của đúng 20 thằng cần thiết, thay vì load full row của 10.020 thằng.

## Các sai lầm thường gặp (Common Pitfalls)

1.  **Lạm dụng Index:** Thấy chậm là add index vô tội vạ. Kết quả là insert/update chậm rì vì DB phải đi update một đống index.
2.  **Tính toán trên cột được Index:**
    Ví dụ: `WHERE YEAR(create_date) = 2024`.
    Viết thế này MySQL không dùng được index của `create_date` đâu. Phải đổi thành: `WHERE create_date BETWEEN '2024-01-01' AND '2024-12-31'`.
3.  **Union vs Union All:** Mặc định `UNION` sẽ lọc trùng (distinct), tốn thêm chi phí sort và so sánh. Nếu chắc chắn dữ liệu không trùng hoặc không quan tâm trùng, hãy dùng `UNION ALL` cho nhẹ đầu.

## Kinh nghiệm thực tế / Lesson Learned

Có lần mình làm tính năng export report cho khách hàng. Query gốc chạy mất 15 phút mới xong, server báo timeout liên tục.
Vấn đề nằm ở chỗ query đó join tới 10 bảng để lấy đủ thông tin hiển thị.
Giải pháp của mình lúc đó không phải là tối ưu câu SQL đó (vì nó quá nát rồi), mà là **Denormalization** (Phi chuẩn hóa). Mình tạo ra một bảng `summary_reports`, định kỳ job chạy ngầm sẽ tổng hợp số liệu và đẩy vào bảng này.
Lúc user export, chỉ cần `SELECT` từ bảng summary, response time giảm từ 15 phút xuống còn... 200ms.
Bài học: Đừng cố ép MySQL làm những việc tính toán phức tạp realtime nếu không cần thiết.

## Kết luận

Tối ưu query là một nghệ thuật cân bằng. Không có công thức chung cho mọi trường hợp, nhưng tư duy đúng đắn là:
1.  Chỉ lấy dữ liệu mình cần.
2.  Hiểu rõ cách MySQL thực thi query (Explain Plan).
3.  Sẵn sàng chia nhỏ bài toán lớn thành nhiều bài toán nhỏ.

Anh em nào muốn đào sâu hơn thì nên đọc kỹ mục "Query Execution Basics" trong chương 8 này, cực cuốn. Chúc anh em tối ưu ngon lành, hệ thống chạy mượt để còn về sớm với vợ con.

**Nguồn:**
- High Performance MySQL, 4th Edition – Chapter 8: Query Performance Optimization[1]
- Official MySQL Documentation