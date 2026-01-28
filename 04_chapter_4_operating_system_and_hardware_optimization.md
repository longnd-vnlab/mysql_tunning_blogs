# [MySQL & OS Tuning] Khi phần cứng là "kẻ giấu mặt" bóp nghẹt performance

## Mở đầu – Tản mạn dev

Mọi người chắc không lạ gì cảnh tượng này: Code đã tối ưu, index đã đánh đầy đủ, query explain ra toàn "const" với "ref" đẹp như mơ, thế mà hệ thống vẫn chạy như rùa bò mỗi khi traffic nhích lên một tí. (sad)

Anh em dev nhà mình thường có thói quen đổ lỗi cho code hoặc query trước tiên. Nhưng đôi khi, vấn đề lại nằm ở tầng thấp hơn mà chúng ta ít để ý: Operating System (OS) và Hardware. Hôm nay mình sẽ chia sẻ lại những điểm cốt lõi từ **Chapter 4: Operating System and Hardware Optimization** của cuốn sách gối đầu giường "High Performance MySQL, 4th Edition". Đảm bảo đọc xong anh em sẽ có cái nhìn khác về con server mình đang chạy, không còn cảnh "thầy bói xem voi" mỗi khi DB lăn ra ốm nữa. (hẹ hẹ)

## Tổng quan

Trong thế giới MySQL, OS và Hardware đóng vai trò như móng nhà. Móng yếu thì nhà đẹp mấy cũng rung lắc. Hiểu về cách MySQL tương tác với CPU, RAM, Disk I/O không chỉ giúp anh em chọn cấu hình server chuẩn (đỡ tốn tiền công ty) mà còn giúp tuning hệ thống chạy mượt mà, ổn định hơn.

Concept cốt lõi ở đây là tìm ra "bottleneck" (điểm nghẽn). Ngày xưa bottleneck thường là Disk I/O (vì HDD chậm rì), nhưng thời nay với sự phổ biến của SSD/NVMe, cuộc chơi đã thay đổi. Bottleneck giờ đây có thể nhảy sang CPU hoặc thậm chí là Network mà anh em không hề hay biết.

Dưới đây là 3 chủ đề kỹ thuật mình thấy "thấm" nhất từ chương này, mời mọi người cùng mổ xẻ.

## 1. Chọn CPU: Latency hay Throughput?

Nhiều anh em khi build server DB cứ auto nghĩ "càng nhiều core càng tốt". Nhưng thực tế nó lằng nhằng hơn thế. Để chọn CPU chuẩn, chúng ta phải hiểu workload của mình cần gì: **Low Latency** (phản hồi nhanh) hay **High Throughput** (xử lý nhiều việc cùng lúc).

- **Low Latency:** Nếu app của bạn cần trả kết quả cực nhanh cho từng query đơn lẻ, thì tốc độ xung nhịp (clock speed) của từng core mới là chân ái. MySQL (cụ thể là các phiên bản cũ hoặc các query phức tạp) thường xử lý một query trên một single thread. Lúc này, con CPU 64 core mà xung nhịp thấp tẹt thì chạy còn thua con 8 core xung nhịp cao.
- **High Throughput:** Ngược lại, nếu hệ thống phải gánh hàng nghìn connection nhỏ li ti cùng lúc (kiểu web e-commerce, social), lúc này số lượng core nhiều lại phát huy tác dụng. Nó giúp MySQL xử lý song song nhiều request mà không bị kẹt thread.

**Lời khuyên:** Đừng chỉ nhìn vào số core. Hãy cân bằng. Nếu budget có hạn, mình ưu tiên con CPU có xung nhịp cao hơn là con CPU nhiều core mà yếu nhớt. Trừ khi bạn chắc chắn workload của mình là concurrency cực cao.

## 2. Ổ cứng SSD và câu chuyện RAID Controller

Thời đại này mà còn chạy DB trên HDD thì đúng là... tội ác (trừ khi lưu log hoặc cold data). SSD đã thay đổi hoàn toàn cuộc chơi database.

Tuy nhiên, có một cái bẫy mà anh em hay dính là **RAID Controller Cache**.
Khi dùng RAID phần cứng, nó thường có một cục pin (BBU - Battery Backup Unit) để nuôi cache khi mất điện.
- **Cơ chế Write-Back:** Dữ liệu được ghi vào cache của card RAID (siêu nhanh) rồi card báo "xong rồi" cho OS, sau đó mới từ từ ghi xuống đĩa thật. Cái này giúp performance ghi tăng chóng mặt (ngon lành).
- **Vấn đề:** Nếu cục pin BBU bị chai hoặc hỏng, card RAID khôn lỏi sẽ tự động chuyển về chế độ **Write-Through** (ghi thẳng xuống đĩa cho an toàn). Lúc này performance ghi sẽ tụt dốc không phanh, có khi chậm đi 10-20 lần mà anh em không hiểu tại sao.

Mình từng gặp vụ này, DB đang chạy ngon tự nhiên slow log bắn ầm ầm. Check hết query không ra, cuối cùng lọ mọ vào check RAID health thì thấy báo "Battery Failed". Thay cục pin phát là lại "bùm", nhanh như gió.

## 3. Tuning Linux Kernel: Filesystem và I/O Scheduler

Đây là phần mà dev thường "ngại" đụng vào nhất vì sợ làm hỏng server, nhưng hiệu quả nó mang lại thì cực ảo ma.

**Về Filesystem:**
Sách recommend mạnh mẽ anh em dùng **XFS** thay vì ext4 cho MySQL. XFS quản lý các file lớn tốt hơn và ít bị lock lằng nhằng hơn khi concurrency cao. Ext4 vẫn ổn, nhưng XFS là lựa chọn tối ưu hơn cho Database Server trên Linux.

**Về I/O Scheduler:**
Mặc định Linux hay để scheduler là `cfq` (Completely Fair Queuing) - cái này tốt cho máy tính cá nhân dùng desktop, nhưng với server DB thì nó là thảm họa. Nó cố gắng công bằng cho các process, dẫn đến việc MySQL bị "xếp hàng" chờ ghi đĩa một cách vô lý.

Anh em nên đổi sang `noop` (cho máy ảo/SSD) hoặc `deadline` (cho HDD truyền thống).
Cách check và đổi nóng (không cần reboot):

```bash
# Check scheduler hiện tại của ổ sda
cat /sys/block/sda/queue/scheduler
# Kết quả: [mq-deadline] kyber bfq none (cái nào nằm trong ngoặc vuông là đang active)

# Đổi sang noop (hoặc none trên kernel mới)
echo none > /sys/block/sda/queue/scheduler
```

Cấu hình này giúp OS bỏ qua các thuật toán sắp xếp request phức tạp, cứ có request là đẩy thẳng xuống cho thiết bị lưu trữ xử lý (vì SSD nó tự lo vụ này quá tốt rồi).

## Các sai lầm thường gặp (Common Pitfalls)

1.  **Swappiness quá cao:** Mặc định Linux để `vm.swappiness = 60`. Khi RAM bắt đầu đầy, OS sẽ hăng hái đẩy bớt RAM của MySQL xuống swap disk (chậm như rùa). Với DB Server, anh em nên set cái này về `0` hoặc `1` . Thà để MySQL chiếm hết RAM còn hơn là bị swap in/out liên tục.
2.  **DNS Lookup:** MySQL mặc định sẽ cố resolve DNS ngược (từ IP ra hostname) cho mỗi connection mới. Nếu DNS server chập chờn, việc connect vào DB sẽ bị delay vài giây đến vài chục giây.
    *Fix:* Thêm `skip-name-resolve` vào `my.cnf`. Đảm bảo user trong MySQL được grant quyền theo IP chứ không theo hostname nhé.

## Kinh nghiệm thực tế / Lesson Learned

Có lần team mình scale hệ thống, RAM server 64GB, mình cấp cho `innodb_buffer_pool_size` tận 58GB (vì tham, muốn cache nhiều). Kết quả là thỉnh thoảng server bị treo cứng, SSH cũng không vào được.

Hóa ra là do **OOM Killer** (Out Of Memory Killer) của Linux. Khi OS thiếu RAM cho các tác vụ hệ thống (kernel, network connection...), nó sẽ tìm thằng nào đang ăn nhiều RAM nhất để "trảm". Và đương nhiên, thằng MySQL béo tốt nhất sẽ lên thớt đầu tiên. (sad)

**Bài học:** Luôn chừa lại khoảng 10-15% RAM (hoặc tối thiểu 2-4GB) cho OS và các process phụ trợ. Đừng bao giờ cấp full RAM cho Database. Ngoài ra, có thể config `oom_score_adj` cho process `mysqld` để dặn OS là "tha cho em nó, giết thằng khác đi" nếu có biến.

## Kết luận

Tối ưu hóa OS và Hardware không phải là phép thuật, nó là sự hiểu biết về cách dòng dữ liệu chảy từ Memory xuống Disk. Thay vì cứ cắm đầu vào `EXPLAIN` query, thi thoảng anh em hãy ngó xuống tầng dưới xem cái nền móng có đang gào thét không nhé.

Tư duy đúng ở đây là: **Balance (Cân bằng)**. Không có cấu hình "best practice" cho mọi trường hợp, chỉ có cấu hình phù hợp nhất với workload hiện tại.

Sắp tới nếu rảnh mình sẽ viết tiếp về các tham số config trong `my.cnf`, chủ đề này cũng nhiều cái hay ho lắm. Happy coding!

**Nguồn:**
- High Performance MySQL, 4th Edition – Chapter 4: Operating System and Hardware Optimization[1]
- Official MySQL Documentation[1]