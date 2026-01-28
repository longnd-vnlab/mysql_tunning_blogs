Dưới đây là bài blog kỹ thuật được viết theo đúng yêu cầu và persona của bạn.

***

# [Deep Dive] Performance Schema: Tấm phim X-Quang soi "nội tạng" MySQL

## Mở đầu – Tản mạn dev

Chắc anh em làm backend/infra ở đây ai cũng từng gặp cái cảnh "ảo ma Canada" này: Hệ thống đang chạy ngon lành, đùng một cái CPU của DB server dựng ngược lên 90%, query thì timeout hàng loạt. Mở slow log ra xem thì... trống trơn hoặc toàn query cơ bản chạy nhanh như điện. Lúc đấy chỉ biết vò đầu bứt tai: "Ủa, rốt cuộc con MySQL nó đang làm cái gì ở trỏng vậy ta?".

Mình từng có lần đi debug một con DB bị treo cứng ngắc, `SHOW PROCESSLIST` thì toàn thấy `Waiting for table metadata lock` mà không biết thằng nào đang giữ lock. Cảm giác lúc đó đúng kiểu thầy bói xem voi, đoán già đoán non rồi restart server cho xong chuyện. (sad)

Đó là lúc anh em cần đến **Performance Schema**. Nếu Slow Query Log chỉ cho anh em biết "cái gì chậm", thì Performance Schema sẽ trả lời câu hỏi "tại sao nó chậm" và "nó đang kẹt ở đâu". Hôm nay mình sẽ mổ xẻ Chapter 3 của cuốn *High Performance MySQL (4th Edition)* để anh em thấy sức mạnh thực sự của công cụ này nhé.

## Tổng quan

Performance Schema (P_S) không phải là cái gì quá mới mẻ, nhưng mình thấy nhiều anh em vẫn ngại dùng vì... nhìn nó lằng nhằng quá. Bảng gì mà tên dài ngoằng, select ra thì toàn số má.

Về bản chất, anh em cứ hình dung P_S như một hệ thống camera giám sát (CCTV) được gắn vào từng ngõ ngách của MySQL code.
- Nó hoạt động ở mức **low-level**: đo đạc từng function call, từng cái lock mutex, từng lần cấp phát bộ nhớ.
- Nó lưu trữ dữ liệu vào các bảng trong memory (storage engine là `PERFORMANCE_SCHEMA`).

Hai concept cốt lõi anh em phải nắm là:
1.  **Instruments (Cảm biến):** Đây là các điểm trong code MySQL được gắn code theo dõi. Ví dụ: "đo thời gian chờ I/O", "đếm số lần query này chạy".
2.  **Consumers (Nơi lưu trữ):** Dữ liệu thu thập được từ Instruments sẽ được ghi vào đâu? Chính là các bảng Consumer. Nếu Instruments là camera, thì Consumer chính là ổ cứng lưu video.

Hiểu đơn giản: Muốn biết gì thì bật Instrument đó lên, và muốn lưu lại để xem thì bật Consumer tương ứng.

***

## 1. Instruments & Consumers: Cấu hình sao cho "ngon"?

Mặc định, MySQL không bật hết tất cả các instruments vì sợ ăn resources (CPU/RAM). Nguyên lý hoạt động của nó như cái phễu lọc vậy.

Trong bảng `setup_instruments`, tên của instrument được tổ chức theo dạng phân cấp, ngăn cách bằng dấu `/`. Ví dụ:
`wait/synch/mutex/innodb/autoinc_mutex`
- `wait`: Loại sự kiện (chờ đợi).
- `synch`: Phân loại con (đồng bộ hóa).
- `mutex`: Loại lock.
- `innodb`: Subsystem.
- `autoinc_mutex`: Tên cụ thể.

Để bật/tắt, anh em có thể update trực tiếp vào bảng này:
```sql
-- Bật tất cả instruments liên quan đến query SELECT
UPDATE performance_schema.setup_instruments
SET ENABLED = 'YES', TIMED = 'YES'
WHERE NAME LIKE 'statement/sql/select';
```

Nhưng lưu ý, bật `Instruments` mà không bật `Consumers` thì cũng như quay phim mà không bỏ băng vào ghi. Anh em cần check bảng `setup_consumers`. Một số consumer quan trọng:
- `events_statements_history`: Lưu lịch sử các câu lệnh SQL vừa chạy.
- `events_waits_current`: Xem hiện tại thread đang chờ cái gì (cực quan trọng để debug lock).
- `events_transactions_history`: Lịch sử transaction (mới có ở các bản sau này).

## 2. Chiến thuật "Bắn tỉa" với Setup Actors & Objects

Một trong những sai lầm kinh điển khi dùng P_S là "bật hết lên cho máu". Kết quả là overhead của việc monitoring làm sập luôn production (hẹ hẹ). Sách *High Performance MySQL* chỉ ra một chiêu rất hay, đó là filter theo đối tượng.

Thay vì monitor toàn bộ server, chúng ta chỉ tập trung vào những "nghi phạm" chính.

### Lọc theo User/Host (Setup Actors)
Bảng `setup_actors` cho phép anh em định nghĩa rule: "Chỉ monitor user `app_main` từ host `10.0.0.5`, còn user `backup` hay `report` thì kệ nó".

```sql
-- Chỉ bật history cho user 'sveta', tắt hết các user khác
INSERT INTO performance_schema.setup_actors
(HOST, USER, ENABLED, HISTORY)
VALUES
('localhost', 'sveta', 'YES', 'YES'), -- Monitor user này
('%', '%', 'NO', 'NO');              -- Mặc định tắt hết
```
Cái này cực hữu dụng khi anh em muốn debug một service cụ thể mà không muốn bị nhiễu bởi các connection khác.

### Lọc theo Object (Setup Objects)
Tương tự, `setup_objects` cho phép anh em chỉ monitor các table hoặc trigger cụ thể.
Ví dụ: Hệ thống có 1000 bảng, nhưng chỉ có bảng `orders` là hay bị lock. Anh em config để P_S chỉ soi kỹ bảng `orders` thôi, đỡ tốn RAM.

## 3. Cỗ máy thời gian: Current - History - History Long

Khi query dữ liệu từ P_S, anh em sẽ thấy các hậu tố `_current`, `_history`, và `_history_long`. Đây là thiết kế dạng **Ring Buffer** (bộ đệm vòng) rất thông minh.

- **`_current`**: Chỉ chứa 1 dòng cho mỗi thread. Nó cho biết "Ngay bây giờ, thread này đang làm cái gì?". Dùng để bắt quả tang realtime.
- **`_history`**: Lưu giữ N sự kiện gần nhất của mỗi thread (mặc định là 10). Dùng để xem "Vừa nãy nó làm gì?".
- **`_history_long`**: Lưu giữ M sự kiện gần nhất **trên toàn bộ server** (bất kể thread nào), mặc định là 10.000. Đây là cái "hộp đen" giúp anh em trace lại sự việc đã trôi qua.

**Mẹo:** Khi debug performance sau sự cố, bảng `events_statements_history_long` là kho báu. Nó lưu lại full stack: query đó chạy mất bao lâu, lock bao lâu, không dùng index nào...

***

## Các sai lầm thường gặp (Common Pitfalls)

1.  **Quên persist cấu hình:** Đây là lỗi "đau đớn" nhất. Anh em hì hục `UPDATE setup_instruments...` chạy ngon lành. Restart DB một cái -> Mất trắng!
    - *Lý do:* Các bảng `setup_` nằm trong memory.
    - *Khắc phục:* Với các biến hệ thống thì dùng `my.cnf`, nhưng với các rule trong `setup_actors` hay `setup_objects`, anh em phải viết script SQL và dùng option `init_file` để load lại mỗi khi khởi động server.

2.  **Ảo tưởng về `TIMED`:** Trong `setup_instruments` có cột `TIMED`. Nếu set `NO`, nó vẫn thu thập events nhưng không đo thời gian (chỉ đếm số lượng). Nhiều anh em tắt `TIMED` để giảm tải nhưng lại thắc mắc sao cột `TIMER_WAIT` toàn số 0.

3.  **Lạm dụng Wildcard:** Chạy lệnh `UPDATE ... SET ENABLED='YES' WHERE NAME LIKE '%'` là cách nhanh nhất để đưa CPU server lên nóc tủ. Chỉ bật những gì mình cần (như `wait/io` hoặc `wait/lock`).

## Kinh nghiệm thực tế / Lesson Learned

Có lần team mình dính vụ DB bị spike CPU định kỳ mỗi 5 phút. Check slow log không thấy gì đặc biệt. Cuối cùng mình mò vào `events_stages_summary_global_by_event_name`.

Thấy cái event `stage/sql/Creating sort index` có số lượng cực lớn và thời gian wait cao bất thường. Hóa ra là do một job background chạy query có `ORDER BY` trên một cột chưa đánh index, nhưng vì nó chạy nhanh (dưới ngưỡng slow log) nhưng tần suất lại quá dày đặc nên làm nghẽn CPU.

**Bài học:** Đừng chỉ nhìn vào những query chạy chậm (long-running). Những query chạy nhanh nhưng tần suất cao (high frequency) mới là những sát thủ thầm lặng. Và Performance Schema là nơi duy nhất vạch mặt được bọn này.

## Kết luận

Performance Schema không phải là công cụ để bật 24/7 ở mức độ chi tiết cao nhất (trừ khi anh em thừa tài nguyên). Tư duy đúng là:
1.  Bật các metrics cơ bản ở mức global.
2.  Khi có sự cố, dùng `setup_actors` / `setup_objects` để khoanh vùng (drill-down).
3.  Kết hợp với **sys schema** (một lớp wrapper dễ đọc hơn của P_S) để đỡ phải nhớ tên bảng lằng nhằng.

Lần tới nếu DB gặp vấn đề, thay vì restart vội, anh em thử bình tĩnh `SELECT * FROM events_waits_current` xem sao nhé. Biết đâu lại tìm ra chân lý!

**Nguồn tham khảo:**
- High Performance MySQL, 4th Edition – Chapter 3: Performance Schema
- Official MySQL Documentation