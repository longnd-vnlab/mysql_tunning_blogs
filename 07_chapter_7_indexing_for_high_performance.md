Giờ mình đã có đủ thông tin từ Chapter 7 để viết bài blog kỹ thuật theo yêu cầu. Mình sẽ viết bài bằng tiếng Việt với phong cách của một technical architect, sử dụng giọng văn thân thiện và có chút humor nhẹ như đã yêu cầu.

***

# Index MySQL: Nghệ Thuật Tối Ưu Hiệu Năng Hay Là Cái Bẫy Tự Đào Cho Mình?

## Mở đầu – Tản mạn dev

Chắc anh em nào làm backend cũng từng trải qua cái cảm giác này: query chạy ngon lành trên dev với vài ngàn rows, lên production với vài triệu rows thì... haizz, timeout tè le. Manager hỏi "sao lại chậm thế?", câu trả lời quen thuộc là "à, để em thêm index". Xong thêm index lung tung, một lúc sau database có cả chục cái index trên cùng một bảng, query vẫn chậm, còn INSERT/UPDATE thì càng ngày càng ì ạch.[1]

Mình từng làm việc với một hệ thống mà team trước để lại – bảng có tận 15 cái index, trong đó 5 cái duplicate nhau, 3 cái chưa bao giờ được dùng, và query vẫn chạy như rùa bò (hẹ hẹ). Đó là lúc mình nhận ra: **index không phải cứ thêm vào là tốt, mà phải hiểu nó hoạt động như thế nào, khi nào cần, khi nào không**.

Bài viết này mình sẽ chia sẻ những kinh nghiệm thực tế về indexing trong MySQL dựa trên Chapter 7 của cuốn High Performance MySQL (4th Edition). Nội dung sẽ giúp anh em hiểu sâu hơn về cách MySQL sử dụng index, làm sao để thiết kế index đúng cách, và quan trọng nhất – tránh được những cái hố mà mình đã từng sa vào.

## Tổng quan

Index trong MySQL giống như mục lục của một cuốn sách – thay vì phải lật từng trang để tìm thông tin, chúng ta nhảy thẳng đến trang cần thiết. Về bản chất, index là một cấu trúc dữ liệu mà storage engine sử dụng để tìm rows nhanh hơn.[1]

Vai trò của index trong MySQL không chỉ đơn thuần là tăng tốc độ query. Nó còn ảnh hưởng đến:
- **Hiệu năng đọc**: Giảm số lượng rows cần scan, tránh full table scan
- **Hiệu năng ghi**: Mỗi index thêm vào là overhead cho INSERT/UPDATE/DELETE[1]
- **Dung lượng lưu trữ**: Index chiếm không gian đáng kể trên disk
- **Memory footprint**: Index được cache vào buffer pool, ảnh hưởng working set

Trong InnoDB – storage engine mà mọi người nên dùng làm default – index được implement dưới dạng B-Tree (chính xác là B+Tree). Điểm đặc biệt là InnoDB tổ chức data theo clustered index, nghĩa là data thực sự được lưu trữ cùng với primary key index. Đây là điểm khác biệt quan trọng so với nhiều database khác và ảnh hưởng trực tiếp đến cách chúng ta thiết kế schema.[1]

## Clustered Index trong InnoDB: Trái Tim Của Storage Engine

### Primary Key chính là clustered index

Khác với nhiều database khác cho phép chọn index nào để cluster, InnoDB tự động cluster data theo primary key. Nếu không define primary key, InnoDB sẽ tìm unique non-nullable index để dùng. Nếu vẫn không có, InnoDB tự tạo một hidden primary key và cluster theo nó – cái này nguy hiểm vì giá trị auto-increment của hidden key được share across tables, gây mutex contention.[1]

Trong clustered index, leaf nodes không chỉ chứa giá trị index mà còn chứa **toàn bộ row data**. Điều này có nghĩa:[1]
- Khi query bằng primary key, chỉ cần một lần lookup là có data
- Related data được lưu gần nhau trên disk (locality of reference)
- Nhưng nếu insert data không theo thứ tự primary key, sẽ gây page split và fragmentation nghiêm trọng

### Secondary index và cái giá phải trả

Tất cả secondary indexes trong InnoDB đều chứa primary key value ở leaf nodes thay vì pointer đến row. Thiết kế này có pros và cons:[1]

**Ưu điểm**: Khi row di chuyển (do update hoặc page split), không cần update secondary index vì primary key không đổi.

**Nhược điểm**: 
- Secondary index lớn hơn nếu primary key lớn
- Lookup qua secondary index cần **hai lần B-tree traversal**: một lần ở secondary index để lấy primary key, một lần nữa ở clustered index để lấy data[1]
- Nếu primary key là UUID hoặc string dài, tất cả secondary indexes đều phình to ra

Mình từng gặp một case bảng users dùng UUID làm primary key. Bảng có 5 secondary indexes, mỗi index đều phải lưu trữ 36-byte UUID. Kết quả là indexes chiếm gần gấp đôi dung lượng so với nếu dùng integer auto-increment. Và performance của lookups qua secondary index cũng chậm hơn đáng kể.[1]

### Insert order và performance

Một trong những bài học đắt giá nhất về clustered index là **insert order matters rất nhiều**. 

Khi insert theo thứ tự tăng dần của primary key (ví dụ auto-increment integer), mọi thứ diễn ra mượt mà: InnoDB append rows vào cuối index, pages được fill gần như tối đa (15/16 full factor), không có page splits, không có fragmentation.[1]

Nhưng khi insert với primary key random (UUID, hash values), thảm họa xảy ra:
- Mỗi row mới phải tìm vị trí random trong index structure
- Gây ra page splits liên tục
- Pages bị sparse và fragmented
- Cuối cùng phải chạy OPTIMIZE TABLE để rebuild

Một benchmark trong sách so sánh insert 3 triệu rows vào hai bảng giống hệt nhau, chỉ khác primary key (integer vs UUID). Kết quả: bảng UUID mất **4525 giây** so với **1233 giây** của integer, và index size lớn hơn 60%. Đây không phải con số đùa được đâu anh em.[1]

## Covering Index: Khi Index Trở Thành Hero

Covering index là một trong những optimization techniques mạnh mẽ nhất mà mình hay sử dụng. Ý tưởng đơn giản: nếu index chứa **tất cả columns** mà query cần, MySQL không cần phải lookup row data.[1]

### Tại sao covering index lại ngon?

Ba lý do chính:
1. **Index entries nhỏ hơn nhiều so với full rows**: Query chỉ cần đọc ít data hơn, quan trọng với cached workloads
2. **Index được sắp xếp**: Range scans trở nên sequential thay vì random IO
3. **Đặc biệt tốt cho InnoDB**: Vì secondary indexes đã chứa primary key, nên thêm vài columns nữa là có thể cover được nhiều queries[1]

Ví dụ thực tế: Bảng `sakila.inventory` có index `(store_id, film_id)`. Query này sẽ được cover hoàn toàn:

```sql
SELECT store_id, film_id FROM sakila.inventory;
```

EXPLAIN sẽ show "Using index" trong Extra column, nghĩa là MySQL không cần đọc table data.[1]

### Mở rộng index để cover query

Một trick mình hay dùng: thay vì tạo index mới, hãy xem có thể extend index hiện có không. Ví dụ có bảng `userinfo` với index trên `state_id`:

```sql
-- Query ban đầu chạy nhanh (chỉ count)
SELECT COUNT(*) FROM userinfo WHERE state_id = 5;  -- 115 QPS

-- Query này chậm vì phải lookup rows
SELECT state_id, city, address FROM userinfo WHERE state_id = 5;  -- 10 QPS
```

Solution: Extend index thành `(state_id, city, address)` để cover query thứ hai. Kết quả: query 2 tăng lên 28 QPS.[1]

Nhưng có một cái bẫy ở đây: sau khi extend index, query 1 chậm đi một chút (từ 115 xuống 100 QPS) vì index lớn hơn. Nếu cả hai queries đều quan trọng, giải pháp là giữ **cả hai indexes** dù có redundant. Trade-off là INSERT sẽ chậm hơn, nhưng đổi lại read performance của cả hai queries đều tốt.[1]

## Column Order trong Composite Index: Nghệ Thuật Hay Khoa Học?

Đây là câu hỏi mình hay gặp nhất: "Nên đặt column nào trước trong composite index?"

### Rule of thumb: Selectivity cao trước

Quy tắc truyền thống là đặt column có selectivity cao (cardinality cao) lên đầu. Selectivity = số distinct values / tổng số rows. Index với selectivity cao filter được nhiều rows hơn.[1]

Ví dụ với query:
```sql
SELECT * FROM payment WHERE staff_id = 2 AND customer_id = 584;
```

Check selectivity:
```sql
SELECT SUM(staff_id = 2), SUM(customer_id = 584) FROM payment;
-- Kết quả: 7992 rows vs 30 rows
```

Theo rule of thumb, nên tạo index `(customer_id, staff_id)` vì customer_id selective hơn.[1]

### Nhưng thực tế phức tạp hơn

Rule of thumb chỉ đúng khi WHERE clause luôn dùng tất cả columns. Trong thực tế:
- Một số queries chỉ dùng prefix của index
- ORDER BY và GROUP BY cũng ảnh hưởng column order
- Data distribution không đều (outliers, skewed data)

Mình từng gặp case một forum có query:
```sql
SELECT COUNT(DISTINCT thread_id) 
FROM Message 
WHERE group_id = 10137 AND user_id = 1288826 AND anonymous = 0;
```

Theo lý thuyết, index `(group_id, user_id)` nên tốt. Nhưng trong thực tế, có một admin user có user_id đặc biệt được add làm friend với **tất cả users** để gửi notifications. User đó có cardinality cực cao, phá vỡ mọi assumptions về selectivity.[1]

Bài học: **Luôn phải kiểm tra data distribution thực tế, đừng tin vào averages**.

### Leftmost prefix rule

MySQL chỉ có thể sử dụng index nếu query match leftmost prefix. Index `(a, b, c)` có thể dùng cho:[1]
- WHERE a = 1
- WHERE a = 1 AND b = 2
- WHERE a = 1 AND b = 2 AND c = 3

Nhưng **không thể** dùng cho:
- WHERE b = 2 (thiếu a)
- WHERE a = 1 AND c = 3 (thiếu b ở giữa)

Điều này có nghĩa column order ảnh hưởng trực tiếp đến những queries nào có thể dùng index. Khi thiết kế, phải cân nhắc tất cả query patterns, không chỉ một query.

## Prefix Index: Tiết Kiệm Dung Lượng Nhưng Đánh Đổi Gì?

Với VARCHAR hoặc TEXT columns dài, MySQL cho phép index chỉ một phần prefix của string thay vì toàn bộ. Điều này tiết kiệm space nhưng giảm selectivity.[1]

### Tính toán prefix length tối ưu

Mục tiêu là tìm prefix đủ dài để selectivity gần với full column, nhưng đủ ngắn để tiết kiệm space. Một kỹ thuật:

```sql
-- Check selectivity của full column
SELECT COUNT(DISTINCT city) / COUNT(*) FROM sakila.city_demo;  -- 0.0312

-- So sánh với prefixes khác nhau
SELECT 
  COUNT(DISTINCT LEFT(city, 3)) / COUNT(*) AS sel3,
  COUNT(DISTINCT LEFT(city, 4)) / COUNT(*) AS sel4,
  COUNT(DISTINCT LEFT(city, 5)) / COUNT(*) AS sel5,
  COUNT(DISTINCT LEFT(city, 6)) / COUNT(*) AS sel6,
  COUNT(DISTINCT LEFT(city, 7)) / COUNT(*) AS sel7
FROM sakila.city_demo;
```

Kết quả cho thấy prefix 7 characters đạt selectivity 0.0310, rất gần với full column (0.0312). Vậy có thể dùng prefix(7) mà không mất nhiều selectivity.[1]

### Nhược điểm của prefix index

Có hai điểm cần lưu ý:
1. MySQL không thể dùng prefix index cho ORDER BY hoặc GROUP BY[1]
2. Prefix index không thể là covering index vì không chứa full value

Do đó prefix index chỉ nên dùng khi:
- Column rất dài và chiếm nhiều space
- Không cần ORDER BY/GROUP BY trên column đó
- Query patterns chủ yếu là equality lookups hoặc LIKE với prefix

## Các Sai Lầm Thường Gặp (Common Pitfalls)

### Lạm dụng tạo quá nhiều single-column indexes

Đây là sai lầm phổ biến nhất mình từng thấy. Developer nghe advice "create indexes on columns in WHERE clause" rồi tạo index cho mọi column:

```sql
CREATE TABLE t (
  c1 INT,
  c2 INT,
  c3 INT,
  KEY (c1),
  KEY (c2),
  KEY (c3)
);
```

Kết quả là MySQL phải dùng index merge strategy – scan nhiều indexes rồi merge kết quả. Đây là suboptimal và thường chậm hơn một composite index được thiết kế tốt.[1]

### Redundant và duplicate indexes

Mình từng audit một database thấy:
```sql
PRIMARY KEY (id),
UNIQUE KEY (id),
INDEX (id)
```

Ba indexes trên cùng một column! Hoàn toàn vô nghĩa nhưng tốn space và làm chậm writes.[1]

Redundant index tinh vi hơn: nếu có index `(a, b)`, thì index `(a)` là redundant vì B-tree index có thể dùng leftmost prefix. Nhưng ngược lại, index `(b)` hoặc `(b, a)` không redundant.[1]

### UUID làm primary key

Đã nói ở trên nhưng nhấn mạnh lại: **đừng dùng UUID làm primary key** trừ khi có lý do thực sự chính đáng. Random values gây:
- Page splits liên tục
- Index fragmentation
- Random disk I/O
- Cache locality kém
- Secondary indexes phình to[1]

Nếu bắt buộc phải dùng UUID (distributed systems, merging data từ nhiều sources), hãy:
- Remove dashes: giảm từ 36 bytes xuống 32 bytes
- Hoặc tốt hơn: convert UUID sang BINARY(16) bằng UNHEX() – chỉ còn 16 bytes[1]
- Cân nhắc dùng UUID version 7 (time-ordered) thay vì version 4 (random)

### Quên update index statistics

MySQL optimizer dựa vào index statistics để quyết định query plan. Statistics không chính xác = bad query plans. InnoDB tính statistics bằng cách sample random pages (mặc định 8 pages).[1]

Statistics được update khi:
- Table lần đầu được open
- Chạy ANALYZE TABLE
- Table size thay đổi đáng kể
- Một số INFORMATION_SCHEMA queries (có thể gây overhead!)[1]

Best practice: Chạy ANALYZE TABLE định kỳ sau khi có bulk data changes, đặc biệt với tables lớn.

### Index fragmentation không được xử lý

Sau nhiều INSERT/UPDATE/DELETE, indexes có thể bị fragmented – pages không đầy, data không sequential trên disk. Điều này giảm performance đáng kể.[1]

Solution: Chạy OPTIMIZE TABLE định kỳ để rebuild table và defragment indexes. Nhưng lưu ý OPTIMIZE TABLE trên InnoDB thực chất là ALTER TABLE ... ENGINE=InnoDB – sẽ rebuild toàn bộ table, lock table, và tốn thời gian với large tables.

## Kinh Nghiệm Thực Tế / Lesson Learned

### Case study 1: Query chậm dù đã có index

Một lần team mình gặp query chậm kinh khủng:
```sql
SELECT * FROM orders 
WHERE customer_id = 12345 AND status = 'pending' 
ORDER BY created_at DESC 
LIMIT 10;
```

Có index `(customer_id, status)` nhưng query vẫn chậm vì phải filesort. EXPLAIN cho thấy "Using filesort".[1]

**Root cause**: Index không match ORDER BY clause. MySQL dùng index để filter nhưng phải sort kết quả.

**Solution**: Đổi index thành `(customer_id, status, created_at)`. Giờ index có thể dùng cho cả filtering lẫn sorting. Performance cải thiện 10x.

**Bài học**: Index phải được thiết kế cho **toàn bộ query** (WHERE + ORDER BY + GROUP BY + SELECT), không chỉ WHERE clause.[1]

### Case study 2: Write performance sụt giảm sau khi thêm indexes

Một bảng `user_activity` với traffic cao (1000+ writes/sec) bắt đầu chậm đi sau khi thêm 3 indexes mới để optimize read queries. Monitoring cho thấy write latency tăng gấp đôi.

**Root cause**: Mỗi INSERT/UPDATE phải maintain tất cả indexes. Benchmark cho thấy với một bảng, thêm indexes tăng INSERT time từ 80s lên 136s cho 1 triệu rows.[1]

**Solution**: 
- Review lại các indexes: 2 trong 3 indexes mới ít được dùng (< 0.1% queries)
- Drop 2 indexes ít dùng
- Với index còn lại, tạo trên replica riêng để serve read traffic, không impact writes

**Bài học**: Indexes không free. Phải cân nhắc read vs write trade-off. Dùng tools như `pt-index-usage` (Percona Toolkit) để track index usage.

### Case study 3: Invisible indexes trong MySQL 8.0

Khi cần drop một index nhưng không chắc liệu có query nào đang dùng, MySQL 8.0 có feature "invisible index" cực kỳ hữu ích:

```sql
ALTER TABLE users ALTER INDEX idx_email INVISIBLE;
```

Index vẫn được maintain nhưng optimizer sẽ ignore nó. Monitor performance vài ngày, nếu không có issues thì drop thật:[1]

```sql
ALTER TABLE users DROP INDEX idx_email;
```

Nếu có vấn đề, make visible lại ngay lập tức mà không tốn thời gian rebuild:

```sql
ALTER TABLE users ALTER INDEX idx_email VISIBLE;
```

**Bài học**: Luôn test impact trước khi drop index. Invisible index là safety net tuyệt vời.

### Lỗi khi scale hệ thống

Khi hệ thống scale từ vài trăm QPS lên vài nghìn QPS, một số queries đột nhiên timeout. Root cause: những indexes "đủ tốt" ở traffic thấp không còn đủ tốt ở traffic cao.

Ví dụ: Index `(created_at)` cho query lấy recent records. Traffic thấp thì OK. Traffic cao, nhiều concurrent queries cùng scan index range, gây mutex contention ở end of B-tree (vì insert order là monotonic increasing).[1]

**Solution**: Shard table theo customer_id hoặc region để phân tán traffic. Hoặc redesign query pattern để tránh scan range lớn.

**Bài học**: Performance testing phải ở scale gần production. Dev environment với ít data không expose được scalability issues.

## Kết luận

Index trong MySQL là double-edged sword – dùng đúng thì performance tăng vọt, dùng sai thì vừa tốn space vừa làm chậm hệ thống. 

**Tư duy đúng khi tiếp cận indexing**:
1. **Hiểu clustered index của InnoDB**: Primary key matters, dùng auto-increment integer thay vì UUID
2. **Design cho cả query**: WHERE + ORDER BY + GROUP BY, không chỉ WHERE
3. **Covering index là best friend**: Đầu tư thời gian để identify covering opportunities
4. **Column order trong composite index**: Cân nhắc selectivity + query patterns + data distribution
5. **Less is more**: Ít indexes tốt hơn nhiều indexes redundant. Quality over quantity
6. **Monitor và maintain**: Chạy ANALYZE TABLE định kỳ, track index usage, xử lý fragmentation

Indexing không phải one-time task mà là ongoing process. Database schema evolve, query patterns thay đổi, data distribution shift – indexes cũng phải evolve theo. Đừng set-and-forget.

**Topic nên tìm hiểu tiếp**:
- Query optimization và EXPLAIN analysis (Chapter 8 của sách này)
- InnoDB internals: buffer pool, adaptive hash index, change buffer
- Index statistics và histogram trong MySQL 8.0
- Tools: pt-query-digest, pt-index-usage, pt-duplicate-key-checker (Percona Toolkit)
- Monitoring index usage với Performance Schema và sys schema

Cuối cùng, nhớ rằng: **premature optimization is the root of all evil**, nhưng ignorance about indexing cũng là root of all slow queries (hẹ hẹ). Hãy tìm balance giữa hai cực này.

Happy indexing, anh em! 🚀

***

**Nguồn:**
- High Performance MySQL, 4th Edition – Chapter 7: Indexing for High Performance[1]
- Official MySQL Documentation
- InnoDB Internal Structure (Jeremy Cole's blog posts)