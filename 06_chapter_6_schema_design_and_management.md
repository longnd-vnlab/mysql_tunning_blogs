# Thiết kế Schema MySQL: Nghệ thuật "nhỏ mà có võ" để scale hệ thống

## Mở đầu – Tản mạn dev

Mọi người chắc hẳn ai cũng từng trải qua cái cảm giác "tim đập chân run" khi gõ lệnh `ALTER TABLE` trên production rồi đúng không? (hẹ hẹ). Kiểu như chỉ sợ bấm Enter xong là database treo cứng, khách hàng la ó, còn sếp thì đứng ngay sau lưng thở dài.

Mình thấy nhiều anh em dev, nhất là mấy bạn mới làm backend, thường ít quan tâm đến việc thiết kế schema kỹ càng từ đầu. Cứ `VARCHAR(255)` cho tiện, hay `TEXT` cho chắc cốp. Đến khi data phình to ra vài chục GB, query chậm như rùa bò thì mới tá hỏa đi tuning. Bài viết này mình chia sẻ lại những đúc kết tâm đắc nhất từ Chapter 6 cuốn **High Performance MySQL, 4th Edition**, hy vọng giúp anh em bớt đau đầu khi hệ thống scale lớn.[1]

## Tổng quan

Schema design trong MySQL không chỉ là vẽ vài cái bảng (table), nối vài cái relationship là xong. Nó là nền tảng của high performance. Một thiết kế tốt giống như móng nhà vững chắc, xây lên bao nhiêu tầng cũng an tâm. Ngược lại, thiết kế ẩu thì query optimization có giỏi mấy cũng chỉ là giải pháp tình thế thôi.

Concept cốt lõi ở đây xoay quanh việc hiểu rõ cách MySQL lưu trữ data trên đĩa và trong memory, từ đó chọn data type tối ưu nhất để giảm thiểu I/O và CPU cycle.[1]

## 1. Nguyên tắc vàng: "Nhỏ hơn thường tốt hơn" (Smaller is usually better)

Đây là thần chú mà anh em architect luôn thuộc nằm lòng. Trong MySQL, data type càng nhỏ gọn thì càng nhanh. Tại sao? Vì nó chiếm ít không gian trên đĩa, ít bộ nhớ RAM và ít CPU cache hơn.[1]

Ví dụ kinh điển là kiểu số nguyên (Integer). Nếu anh em lưu trạng thái (status) chỉ có vài giá trị (0, 1, 2), đừng dùng `INT` (4 bytes), hãy dùng `TINYINT` (1 byte). Nghe có vẻ tiết kiệm vặt vãnh, nhưng khi bảng có hàng trăm triệu dòng, sự khác biệt về dung lượng lưu trữ và tốc độ index là cực lớn.

Một case thú vị khác là lưu địa chỉ IP. Rất nhiều anh em dùng `VARCHAR(15)` để lưu IPv4 (ví dụ '192.168.1.1'). Thực tế, IPv4 bản chất là số nguyên 32-bit không dấu. Hãy dùng `INT UNSIGNED` và hàm `INET_ATON()` / `INET_NTOA()` để chuyển đổi. Vừa tiết kiệm space, vừa index nhanh hơn nhiều so với chuỗi.[1]

> **Tip:** Đừng đánh giá thấp khoảng giá trị (range). `TINYINT UNSIGNED` lưu được từ 0-255, quá đủ cho trường `age` hay `status`. Đừng "phóng tay" dùng `BIGINT` cho những thứ nhỏ bé, trừ khi anh em đang đếm số tiền của Elon Musk (hẹ hẹ).

## 2. Cạm bẫy của chuỗi và kiểu JSON

Xử lý chuỗi (String) trong MySQL khá "lằng nhằng" nếu anh em không để ý.
Hai món thường gặp nhất là `VARCHAR` và `CHAR`.
- `VARCHAR`: Tiết kiệm không gian lưu trữ vì nó chỉ dùng đúng độ dài cần thiết + 1 hoặc 2 byte độ dài. Tuy nhiên, nó dễ gây ra phân mảnh (fragmentation) khi update làm độ dài dòng thay đổi.[1]
- `CHAR`: Độ dài cố định, nhanh hơn khi xử lý vì ít phân mảnh, nhưng lại tốn dung lượng nếu data ngắn hơn khai báo.

Dùng cái nào? Với những thứ độ dài ít đổi như mã hash password (MD5, SHA), `CHAR` là chân ái. Còn tên người dùng, địa chỉ email thì cứ `VARCHAR` mà triển.

**Về JSON:**
Thời nay làm API trả về JSON nhiều, anh em hay tiện tay tống nguyên cục JSON vào cột kiểu `JSON` trong MySQL cho nhanh (schemaless mà lị). Nhưng coi chừng "ăn hành".
Sách đã benchmark thử: cùng một dữ liệu, lưu bằng bảng quan hệ (cột dòng đàng hoàng) chỉ tốn 3 page (16KB/page), trong khi lưu kiểu JSON tốn tới 5 page. Query trên JSON cũng chậm hơn query cột thường dù MySQL 8.0 đã support khá tốt. Chỉ nên dùng JSON cho những data thuộc tính linh động, ít khi dùng để filter hay join thôi nhé.[1]

## 3. Quản lý Schema Change: Ác mộng mang tên ALTER TABLE

Đây là phần "xương xẩu" nhất mà chương này đề cập. Thay đổi schema (thêm cột, đổi type) trên bảng lớn là một cực hình vì nó có thể lock table, block mọi thao tác ghi (write) làm sập service (sad).

MySQL 8.0 đã có cải tiến với `INSTANT` algorithm cho một số thao tác (như thêm cột ở cuối bảng), giúp việc này diễn ra tức thì. Tuy nhiên, không phải cái gì cũng `INSTANT` được.[1]

Với các hệ thống lớn, anh em nên làm quen với các công cụ external như `pt-online-schema-change` (của Percona) hoặc `gh-ost` (của GitHub).[1]
- **Cơ chế:** Các tool này tạo ra một bảng mới (ghost table), copy dữ liệu từ bảng cũ sang, đồng thời sync các thay đổi (write) mới vào bảng này. Cuối cùng nó đổi tên bảng cái "bụp" (atomic rename).
- **Điểm khác biệt:** `pt-online-schema-change` dùng Triggers (dễ gây overhead), còn `gh-ost` đọc Binary Log (an toàn hơn, ít ảnh hưởng performance hơn).[1]

## Các sai lầm thường gặp (Common Pitfalls)

1.  **Lạm dụng NULL**: Nhiều anh em cứ để default là `NULL` cho dễ tính. Nhưng trong MySQL, cột cho phép NULL tốn thêm dung lượng lưu trữ và làm index phức tạp hơn. Tốt nhất là `NOT NULL` và set default value (như 0 hoặc chuỗi rỗng) nếu có thể.[1]
2.  **Quá nhiều cột (Too many columns)**: Đừng tham lam tạo bảng vài trăm cột. MySQL phải tốn CPU để decode row buffer thành các cột mỗi khi đọc. Vài trăm cột là CPU khóc thét đấy.[1]
3.  **Dùng ENUM cho list hay thay đổi**: `ENUM` khá gọn nhẹ, nhưng mỗi lần muốn thêm một giá trị mới vào list (ví dụ thêm trạng thái đơn hàng), anh em lại phải `ALTER TABLE`. Nếu cái list này đổi liên xoành xoạch thì đúng là tự bắn vào chân.[1]

## Kinh nghiệm thực tế / Lesson Learned

Có lần mình join vào dự án dùng `gh-ost` để migrate một bảng cực lớn (vài TB). Mọi thứ setup ngon lành, nhưng khi chạy thì fail sấp mặt. Lý do? Bảng đó dùng **Foreign Key**.
`gh-ost` và nhiều tool online schema change rất "dị ứng" với Foreign Key. Nó làm quy trình phức tạp và rủi ro hơn nhiều. Bài học xương máu: Ở scale lớn, tốt nhất là **bỏ Foreign Key** trong DB đi, hãy validate ràng buộc dữ liệu ở tầng Application. Vừa dễ sharding sau này, vừa đỡ đau đầu khi migrate data. Nghe hơi ngược đời so với lý thuyết đại học, nhưng thực tế nó phũ phàng vậy đó (haizz).[1]

## Kết luận

Thiết kế Schema trong MySQL không phải là việc làm một lần rồi quên. Nó là quá trình cân nhắc đánh đổi (trade-off) liên tục giữa tính toàn vẹn dữ liệu, tốc độ query và khả năng bảo trì.
Tư duy đúng đắn là: **Giữ mọi thứ đơn giản và nhỏ gọn nhất có thể**. Đừng cố nhồi nhét logic phức tạp vào database.

Topic nên tìm hiểu tiếp:
- Indexing Strategies (Chapter 7 - Chương tiếp theo của cuốn sách này, cực hay).
- Query Performance Optimization.
- Partitioning & Sharding.

Happy coding nhé anh em!

**Nguồn tham khảo:**
-  High Performance MySQL, 4th Edition – Chapter 6: Schema Design and Management[1]
-  Official MySQL Documentation (được nhắc đến trong sách)[1]