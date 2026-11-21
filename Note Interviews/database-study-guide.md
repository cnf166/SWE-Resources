# SQL, QUERY OPTIMIZATION, JPA/HIBERNATE, POSTGRESQL, MYSQL & DATABASE INTERNALS
## Tài liệu đề cương học tập và ôn phỏng vấn

Đề cương chi tiết theo mô hình 3 tầng: **Core Internals → Production Engineering → Interview & System Reasoning**

- **Đối tượng:** người học Backend (Java/Spring) hướng tới Intern/Junior và muốn có nền tảng database đủ sâu để tiếp tục phát triển.
- **Phạm vi:** danh sách chủ đề và câu hỏi tự luyện; **không có đáp án**.
- **Nguồn nền tảng:** *High Performance MySQL, 3rd Edition* (Schwartz, Zaitsev, Tkachenko) cho phần MySQL/InnoDB, indexing, query optimization, replication, backup; phần PostgreSQL và Spring Data JPA/Hibernate được bổ sung theo cùng khung học. Các chương của sách được ghi chú dạng *(HPM Ch.X)* để bạn tra cứu.
- **Gợi ý phiên bản:** sách viết cho MySQL 5.1/5.5 (có nhắc 5.6). Nhiều hành vi đã thay đổi ở MySQL 8.x (query cache bị gỡ, optimizer có hash join, window function, CTE, histogram, atomic DDL...). **Luôn đối chiếu với phiên bản thực tế** của PostgreSQL (15/16+), MySQL (8.x), Hibernate (6.x) và Spring Boot (3.x) khi học.

> Mục tiêu không phải học thuộc câu trả lời. Mục tiêu là giải thích được cơ chế, **kiểm chứng bằng EXPLAIN/benchmark/log SQL**, và bảo vệ được lựa chọn kỹ thuật.

---

## Mục lục

- Phần I. Phương pháp luyện trả lời theo khung năm bước
- Phần II. Cách sử dụng tài liệu, mô hình 3 tầng và kế hoạch học
- Phần III. SQL & Schema Design
- Phần IV. Indexing & Query Optimization
- Phần V. Database Internals (dưới "under the hood")
- Phần VI. MySQL & InnoDB
- Phần VII. PostgreSQL
- Phần VIII. Spring Data JPA & Hibernate
- Phần IX. Bài tập cốt lõi và kế hoạch luyện phỏng vấn
- Phần X. Ngân hàng 120 câu hỏi
- Phần XI. Checklist tự đánh giá

---

# PHẦN I — PHƯƠNG PHÁP LUYỆN TRẢ LỜI

Dùng khung năm bước cho mỗi câu hỏi. Tự trả lời trước khi xem tài liệu. Câu đơn giản có thể trả lời ngắn; câu về transaction, MVCC, locking, query plan cần giải thích sâu hơn.

1. **Definition — Khái niệm là gì?** Định nghĩa chính xác, mục đích, vấn đề giải quyết; phân biệt với khái niệm gần giống (vd: isolation vs propagation, clustered vs covering index).
2. **Mechanism — Hoạt động như thế nào?** Luồng xử lý, thành phần tham gia: parser → optimizer → executor → storage engine; buffer pool, WAL/redo log, undo log, lock manager, version chain...
3. **Example — Ví dụ thực tế hoặc code/SQL minh họa.** Có cả trường hợp đúng và trường hợp dễ hiểu nhầm. Nếu được, chạy `EXPLAIN`/`EXPLAIN ANALYZE` hoặc viết test để kiểm chứng.
4. **Limitation — Khi nào không hoạt động như mong đợi?** Edge case, giả định bị phá vỡ, hành vi phụ thuộc phiên bản/engine/cấu hình (vd: index không dùng được khi bọc cột trong hàm).
5. **Trade-off — Khi nào nên/không nên dùng?** So sánh phương án thay thế; chi phí về ghi, dung lượng, độ phức tạp, khả năng bảo trì.

### Mẫu ghi chú cho từng câu hỏi

| Trường | Ghi chú của người học |
|---|---|
| Câu hỏi / Chủ đề | Ghi chính xác câu hỏi và nhóm kiến thức |
| Definition | 2–4 câu bằng lời của bản thân |
| Mechanism | Các bước, thành phần tham gia, sơ đồ nếu cần |
| Example | SQL/code/EXPLAIN output hoặc tình huống thực tế |
| Limitation | Ít nhất một edge case hoặc lỗi thường gặp |
| Trade-off | So sánh tối thiểu một phương án thay thế |
| Mức tự tin | 0 = chưa hiểu; 1 = nhận diện; 2 = giải thích được; 3 = áp dụng/debug được |
| Ngày ôn lại | Ngày cần trả lời lại mà không xem ghi chú |

### Quy tắc tự đánh giá

- Không tính là đã học xong nếu chỉ nhớ định nghĩa mà không giải thích được cơ chế.
- Mọi khẳng định về performance phải có **số đo** (thời gian, rows examined, buffer hits), không dựa cảm giác. Đây là tinh thần của HPM Ch.3: *performance = response time*, "không thể tối ưu thứ không đo được".
- Phân biệt điều **SQL standard bảo đảm** với hành vi **phụ thuộc DBMS** (MySQL vs PostgreSQL), engine (InnoDB vs MyISAM), cấu hình, phiên bản.
- Cảnh giác với "tuning by ratio", công thức thần kỳ, lời khuyên trên mạng không kiểm chứng (HPM Ch.8).

---

# PHẦN II — CÁCH SỬ DỤNG TÀI LIỆU

## Mô hình 3 tầng

| Tầng | Mục tiêu | Câu hỏi tự kiểm tra |
|---|---|---|
| **Tầng 1 — Core Internals** | Hiểu cấu trúc dữ liệu, storage, transaction, optimizer và cơ chế nền tảng | Vì sao nó hoạt động như vậy? Thành phần nào tham gia? |
| **Tầng 2 — Production Engineering** | Áp dụng vào hệ thống thật: tải cao, lỗi, replication, backup, đo đạc | Dưới tải, lỗi, nhiều instance, dữ liệu lớn thì còn đúng không? |
| **Tầng 3 — Interview & System Reasoning** | Giải thích, debug, so sánh phương án, bảo vệ quyết định | Tôi trình bày rõ, có ví dụ, nêu trade-off được không? |

## Nhịp học gợi ý

- **Mỗi tuần:** một nhóm chủ đề chính; đọc tài liệu chính thức; ghi thuật ngữ.
- **Mỗi ngày:** học sâu một phần nhỏ, chạy truy vấn thật trên dataset (Sakila, `employees`, hoặc dữ liệu tự sinh), trả lời 5–10 câu không nhìn tài liệu.
- **Cuối tuần:** mock interview 30–60 phút (tự ghi âm hoặc nhờ người hỏi).
- **Sau mock:** viết lại câu sai/thiếu cơ chế.
- **Mỗi 2–3 tuần:** ôn lại chủ đề cũ.

## Thứ tự học đề xuất (8–10 tuần)

| Tuần | Trọng tâm |
|---|---|
| 1 | SQL nền tảng, JOIN, aggregate, subquery, window function, schema/data types |
| 2 | Index: B-Tree, composite, covering, selectivity, cách đọc EXPLAIN |
| 3 | Query optimization: access types, join strategy, sort, pagination, anti-pattern |
| 4 | Transaction, ACID, isolation, MVCC, locking, deadlock |
| 5 | Storage internals: page, buffer pool, WAL/redo/undo, checkpoint, crash recovery |
| 6 | MySQL/InnoDB chuyên sâu: clustered index, secondary index, replication, backup |
| 7 | PostgreSQL chuyên sâu: heap, MVCC tuple, VACUUM, planner, index types, WAL |
| 8 | Spring Data JPA/Hibernate: entity lifecycle, fetching, N+1, transaction, locking |
| 9 | Tích hợp: profiling, benchmark, scaling, HA, thiết kế case study |
| 10 | Mock interview tổng hợp, ôn câu sai |

## Tài liệu tham khảo chính

- *High Performance MySQL, 3rd ed.* (sách nền tảng của đề cương này)
- MySQL Reference Manual (InnoDB Locking and Transaction Model, EXPLAIN Output Format, Optimization)
- PostgreSQL Documentation (Chapter: Concurrency Control, Indexes, Performance Tips, WAL, Routine Vacuuming)
- *Use The Index, Luke!* (use-the-index-luke.com)
- Hibernate ORM User Guide; Spring Data JPA Reference; Vlad Mihalcea's blog (hibernate performance)
- *Database Internals* (Alex Petrov); *Designing Data-Intensive Applications* (Kleppmann)

> Không cần đọc hết. Dùng tài liệu có cấu trúc trước, sau đó tra manual/source khi cần xác minh một hành vi cụ thể.

---

# PHẦN III — SQL & SCHEMA DESIGN

## Tầng 1 — SQL Language & Relational Model

**Mục tiêu:** viết SQL đúng, dự đoán được kết quả và thứ tự thực thi logic.

### 1. Relational model & SQL căn bản
- Relation, tuple, key (primary, candidate, foreign, surrogate vs natural); ràng buộc toàn vẹn.
- `NULL` và three-valued logic; `NULL` trong `WHERE`, `JOIN`, `GROUP BY`, `COUNT(col)` vs `COUNT(*)` *(HPM Ch.6)*.
- Thứ tự thực thi logic: `FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`.
- `DISTINCT` vs `GROUP BY`; `UNION` vs `UNION ALL` *(chi phí temp table + dedup, HPM Ch.6)*.

### 2. JOIN & subquery
- `INNER/LEFT/RIGHT/FULL/CROSS JOIN`; semi-join (`EXISTS`/`IN`) và anti-join (`NOT EXISTS`/`LEFT JOIN ... IS NULL`/`NOT IN` và bẫy `NULL`).
- Correlated vs uncorrelated subquery; derived table; CTE (và khác biệt materialized/inline theo DBMS, phiên bản).
- Join decomposition: khi nào tách join thành nhiều query ở application *(HPM Ch.6)*.
- Self-join, join nhiều bảng và giới hạn (MySQL 61 tables/join; planning cost tăng giai thừa).

### 3. Aggregate, window function, nâng cao
- `GROUP BY` với cột không nằm trong group (`ONLY_FULL_GROUP_BY`) *(HPM Ch.6, Ch.8)*; `HAVING` vs `WHERE`; `GROUP BY ... WITH ROLLUP`.
- Window function: `ROW_NUMBER/RANK/DENSE_RANK/LAG/LEAD/SUM() OVER`, frame (`ROWS` vs `RANGE`).
- Top-N per group, running total, ranking: so sánh window function với user-defined variable trick *(HPM Ch.6)* và subquery.
- `INSERT ... ON DUPLICATE KEY UPDATE` (MySQL) vs `INSERT ... ON CONFLICT` (PostgreSQL); `MERGE`.

### 4. DDL, data types, schema design *(HPM Ch.4)*
- Chọn data type: nhỏ hơn thường tốt hơn; đơn giản tốt hơn; tránh `NULL` khi không cần.
- Số nguyên (`INT(11)` không giới hạn giá trị), `DECIMAL` vs `FLOAT/DOUBLE`, lưu tiền bằng `DECIMAL`/`BIGINT` đơn vị nhỏ nhất.
- Chuỗi: `CHAR` vs `VARCHAR` (length prefix, trailing space), `TEXT/BLOB` (off-page, temp table on disk), `ENUM`.
- Thời gian: `DATETIME` vs `TIMESTAMP` (phạm vi, time zone, 2038), lưu timezone đúng cách.
- Định danh: AUTO_INCREMENT/sequence vs UUID (insert ngẫu nhiên, page split, `BINARY(16)`/UUIDv7), IP bằng số nguyên.
- Chuẩn hóa (1NF–BCNF) vs phi chuẩn hóa có chủ đích; summary/cache table, counter table, materialized view *(HPM Ch.4)*.
- Anti-pattern: EAV, "ENUM in disguise", quá nhiều cột, autogenerated schema/ORM schema *(HPM Ch.4)*.
- Foreign key: lợi ích (toàn vẹn) vs chi phí (lock, index, ghi) *(HPM Ch.7)*.

## Tầng 2 — Production SQL & Schema

- Schema migration an toàn: online DDL, shadow table + rename, công cụ như `pt-online-schema-change`/`gh-ost`, lock metadata *(HPM Ch.4)*.
- Soft delete, audit/history table, partition theo thời gian, archive/purge theo lô nhỏ *(HPM Ch.6, Ch.11)*.
- Thiết kế bảng counter, queue table: tránh `SELECT ... FOR UPDATE`, claim bằng `UPDATE` *(HPM Ch.6 case study)*; `FOR UPDATE SKIP LOCKED`.
- Xử lý tiền, thời gian, múi giờ, character set/collation (`utf8` vs `utf8mb4`), so sánh case-insensitive *(HPM Ch.7)*.
- Kiểm soát `sql_mode`, strict mode, nhập dữ liệu rác.
- Tránh `SELECT *`, fetch quá nhiều row/column, truy vấn lặp lại cùng dữ liệu *(HPM Ch.6)*.

## Tầng 3 — Interview & Problem Solving

- Viết SQL trên giấy/bảng: top-N per group, gaps and islands, running total, tìm bản ghi trùng, xóa trùng giữ lại một bản.
- Code reasoning: dự đoán kết quả với `NULL`, `NOT IN` + `NULL`, `LEFT JOIN` + điều kiện ở `WHERE` vs `ON`.
- Thiết kế schema cho: hệ thống đặt hàng, đăng ký tín chỉ, POS nhà hàng, social feed, queue công việc.
- Nêu giả định, ràng buộc, truy vấn chính trước khi chốt schema.

### Bài tập thực hành SQL & schema
1. Cài Sakila; viết 15 truy vấn có `JOIN`, `GROUP BY`, `HAVING`, subquery, window function.
2. Viết cùng một truy vấn "phim không có diễn viên" bằng `NOT EXISTS`, `LEFT JOIN ... IS NULL`, `NOT IN`; so sánh kết quả khi có `NULL`.
3. Cài đặt top-3 sản phẩm theo doanh thu cho từng danh mục bằng window function và bằng subquery.
4. Thiết kế schema đăng ký tín chỉ (sinh viên, lớp học phần, đăng ký, ràng buộc sĩ số); nêu index cần có.
5. Thiết kế bảng lưu tiền và thời gian đúng; viết test chứng minh `FLOAT` sai số.
6. Cài đặt queue table bằng `UPDATE ... LIMIT` claim; thử với nhiều worker song song.

---

# PHẦN IV — INDEXING & QUERY OPTIMIZATION

## Tầng 1 — Cơ chế index và optimizer

### 1. Index cơ bản *(HPM Ch.5)*
- Mục đích index: giảm dữ liệu phải đọc, tránh sort/temp table, đổi random I/O thành sequential I/O.
- B-Tree/B+Tree: leaf, node, cấu trúc sắp thứ tự, range scan, `ORDER BY` nhờ index.
- Leftmost prefix, cột thứ tự trong composite index, không "nhảy cột", range condition chặn cột phía sau.
- Hash index (Memory engine; adaptive hash index của InnoDB; tự dựng pseudo-hash bằng CRC32), full-text index, spatial (R-Tree).
- **Selectivity & cardinality**; prefix index (chọn độ dài prefix); worst-case selectivity (giá trị "đặc biệt" như guest user).
- Nguyên tắc 3-star (Lahdenmaki & Leach): rows kề nhau, đúng thứ tự, đủ cột (covering).

### 2. Clustered vs secondary index *(HPM Ch.5)*
- Clustered index (InnoDB): row nằm ở leaf của primary key; secondary index chứa PK làm "con trỏ" → hai lần B-Tree lookup, secondary index phình khi PK lớn.
- So sánh với MyISAM (non-clustered, row pointer); với PostgreSQL (heap + index trỏ ctid).
- Chèn theo thứ tự PK vs ngẫu nhiên (UUID): page split, fragmentation, fill factor.
- Covering index: `Using index`; InnoDB "extra columns" nhờ PK trong secondary index; deferred join.

### 3. Query execution & optimizer *(HPM Ch.6)*
- Vòng đời query: client → parse → preprocess → optimize → execute → trả kết quả (streaming).
- Cost-based optimizer: nguồn thống kê, ước lượng cardinality, sai số, vì sao chọn plan "sai".
- Nested-loop join, join order, `STRAIGHT_JOIN`; hash join/merge join (PostgreSQL, MySQL 8); search space giai thừa.
- Sort: filesort in-memory vs on-disk, single-pass vs two-pass algorithm; `max_length_for_sort_data`.
- Temp table: implicit temporary table, điều kiện chuyển xuống disk (`tmp_table_size`, BLOB/TEXT).
- Limitation của optimizer MySQL cũ: correlated subquery `IN()`, UNION không push-down, loose index scan, `MIN/MAX`, `SELECT` + `UPDATE` cùng bảng.

### 4. Đọc EXPLAIN *(HPM Appendix D)*
- Các cột: `id`, `select_type`, `table`, `type` (ALL, index, range, ref, eq_ref, const/system, NULL), `possible_keys`, `key`, `key_len`, `ref`, `rows`, `filtered`, `Extra`.
- `Extra` quan trọng: `Using index`, `Using where`, `Using temporary`, `Using filesort`, `Using index condition`.
- `EXPLAIN` ước lượng ≠ thực thi thật: dùng `EXPLAIN ANALYZE` (MySQL 8/PostgreSQL), `SHOW STATUS`/handler counters *(HPM Ch.3)*.
- PostgreSQL: `EXPLAIN (ANALYZE, BUFFERS)`, Seq Scan / Index Scan / Index Only Scan / Bitmap Heap Scan, Nested Loop / Hash Join / Merge Join, actual vs estimated rows.

## Tầng 2 — Query Optimization trong Production

### 1. Phương pháp (response-time oriented) *(HPM Ch.3)*
- Đo trước, đổi sau: profile ở tầng application → server → query.
- Slow query log (`long_query_time=0` có kiểm soát), `pt-query-digest`, profile: rank theo tổng response time, V/M, histogram.
- Tránh tối ưu thứ không đáng (Amdahl's Law); tối ưu outlier; chú ý lost time.
- Intermittent problems: trigger rõ ràng, thu thập diagnostic data, phân biệt cause vs effect *(case study "cache stampede")*.
- PostgreSQL: `pg_stat_statements`, `auto_explain`, `pg_stat_activity`, log_min_duration_statement.

### 2. Kỹ thuật tối ưu query
- Isolate cột (không bọc cột trong hàm/biểu thức: `WHERE actor_id + 1 = 5`, `DATE(col)`), tránh implicit cast.
- Tối ưu `COUNT()`, `LIMIT/OFFSET` (keyset/seek pagination, deferred join), `SQL_CALC_FOUND_ROWS`, `GROUP BY/DISTINCT`, `UNION ALL`.
- Chọn thứ tự cột composite index theo query thực tế, không theo quy tắc "most selective first" mù quáng.
- Range + equality: dùng `IN()` list để mở rộng prefix; tránh nhiều range condition *(HPM Ch.5 case study)*.
- Loại index trùng lặp/thừa/không dùng; chi phí duy trì index với ghi.
- Rewrite subquery ↔ join, `EXISTS`; query hints và rủi ro khi nâng cấp phiên bản.
- Bảo trì: `ANALYZE TABLE`/statistics, fragmentation, `OPTIMIZE TABLE`; PostgreSQL `ANALYZE`, `VACUUM`, `REINDEX`.

### 3. Application-level *(HPM Ch.14)*
- Quá nhiều/quá ít query; connection pooling vs persistent connection; query "rác" (ping, `SET NAMES`).
- Caching nhiều tầng: TTL, explicit invalidation, invalidation on read, cache object hierarchy, cache stampede.
- Pregenerate nội dung; chọn đúng công cụ (search engine ngoài, Redis, queue) khi DB không phù hợp.

## Tầng 3 — Interview & Reasoning

- Cho một query chậm + `EXPLAIN` → chẩn đoán và đề xuất index/rewrite.
- Cho schema + bộ query → thiết kế bộ index tối thiểu; giải thích vì sao không thêm index khác.
- Giải thích vì sao index "có mà không dùng" (selectivity thấp, hàm bọc cột, collation/type mismatch, stats cũ, optimizer chọn full scan rẻ hơn).
- So sánh offset pagination với keyset pagination trên bảng 100 triệu dòng.

### Bài tập thực hành
1. Tạo bảng 5–10 triệu dòng; đo truy vấn với/không có index; ghi `rows examined`, thời gian.
2. Chứng minh leftmost prefix: tạo `(a,b,c)`, thử `WHERE b=?`, `WHERE a=? AND c=?`, `WHERE a=? AND b>? AND c=?`.
3. Làm lại case study dating-site indexing *(HPM Ch.5)* với `IN()` trick và cột `active` precomputed.
4. So sánh `LIMIT 100000,20` với deferred join và keyset pagination; vẽ biểu đồ thời gian.
5. Viết slow query log mẫu và phân tích bằng `pt-query-digest`; xếp hạng query cần tối ưu.
6. So sánh hash index tự dựng (`CRC32(url)`) với B-Tree index trên cột URL dài; kiểm tra collision.
7. Tạo ra truy vấn khiến optimizer chọn sai join order; sửa bằng index/hint/rewrite và ghi nhận trade-off.

---

# PHẦN V — DATABASE INTERNALS ("UNDER THE HOOD")

Phần này độc lập DBMS cụ thể, sau đó đối chiếu MySQL/InnoDB và PostgreSQL.

## Tầng 1 — Core Internals

### 1. Kiến trúc DBMS
- Phân tầng: connection/auth → parser → rewriter → optimizer → executor → storage engine → buffer manager → disk.
- MySQL: server layer + pluggable storage engine API *(HPM Ch.1)*; PostgreSQL: process-per-connection, shared buffers, background processes (checkpointer, bgwriter, WAL writer, autovacuum).
- Mô hình thread-per-connection (MySQL) vs process-per-connection (PostgreSQL) và hệ quả với số kết nối, pooling (PgBouncer/HikariCP).

### 2. Storage & page
- Page/block (InnoDB 16 KB, PostgreSQL 8 KB), row format, free space, fill factor, overflow/off-page (TOAST, BLOB).
- Heap file vs index-organized table; tablespace; file-per-table *(HPM Ch.8)*.
- Fragmentation: row, intra-row, free-space fragmentation *(HPM Ch.5)*.
- Row-oriented vs column-oriented; LSM-tree vs B-Tree (khái niệm), vì sao OLTP chọn B-Tree.

### 3. Buffer pool & caching
- Buffer pool/shared buffers: page cache, LRU (young/old sublist), dirty page, flush list, read-ahead.
- Double buffering với OS page cache; `O_DIRECT`; working set, cache miss rate, memory-to-disk ratio *(HPM Ch.9)*.
- Doublewrite buffer, partial page write, checksum.

### 4. Logging & recovery
- **WAL (write-ahead logging)**: ghi log tuần tự trước, flush data page sau; `fsync`, group commit *(HPM Ch.1, Ch.8)*.
- InnoDB: redo log (log file size, log buffer, `innodb_flush_log_at_trx_commit`), undo log, binary log (logical, cho replication/PITR), 2PC giữa binlog và InnoDB (`innodb_support_xa`, `sync_binlog`).
- PostgreSQL: WAL (LSN, segment), checkpoint, full-page writes, `wal_level`, `synchronous_commit`.
- Crash recovery: redo (roll-forward), undo (rollback uncommitted), checkpoint, "furious flushing".
- Durability levels: write vs flush; battery-backed write cache; fake `fsync`.

### 5. Transaction & concurrency control
- ACID; ghi rõ cái gì bảo đảm Atomicity (undo), Durability (redo/WAL), Isolation (lock/MVCC), Consistency.
- **Isolation levels**: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE; dirty read, non-repeatable read, phantom read, lost update, write skew.
- Hành vi khác nhau theo DBMS: MySQL mặc định REPEATABLE READ (next-key lock chặn phantom), PostgreSQL mặc định READ COMMITTED (REPEATABLE READ = snapshot isolation; SERIALIZABLE = SSI).
- **MVCC**: version chain (InnoDB undo + hidden `trx_id`/roll pointer) vs tuple versioning trong heap (PostgreSQL `xmin/xmax`); snapshot/read view; consistent nonlocking read *(HPM Ch.1)*.
- Locking: shared/exclusive, row lock, table lock, intention lock, record/gap/next-key lock, metadata lock, `SELECT ... FOR UPDATE/FOR SHARE`, two-phase locking *(HPM Ch.1, Appendix E)*.
- Deadlock: nguyên nhân, detect (wait-for graph), victim selection, cách thiết kế để giảm deadlock (thứ tự khóa nhất quán), retry.
- Optimistic vs pessimistic concurrency control.

### 6. Query processing nội bộ
- Parse tree, rewrite, plan tree (left-deep tree trong MySQL), cost model, statistics (histogram, n_distinct, MCV).
- Access method: seq scan, index scan, index-only, bitmap scan; join algorithm: nested loop, hash, merge.
- Sort & aggregate: in-memory vs external merge sort; hash aggregate vs group aggregate.
- Prepared statement: static vs dynamic optimization, plan cache, generic vs custom plan *(HPM Ch.7)*.

### 7. Replication & distribution (khái niệm)
- Statement-based vs row-based vs mixed replication; asynchronous/semisynchronous/synchronous.
- Binary log/relay log; PostgreSQL streaming replication (physical), logical replication; replication lag; single-threaded apply bottleneck *(HPM Ch.10)*.
- CAP/PACELC mức khái niệm; consensus (Galera/Paxos/Raft) mức nhận biết *(HPM Ch.12)*.

## Tầng 2 — Production Behavior & Reliability

- **Capacity & scaling** *(HPM Ch.11)*: scale up vs scale out; read replica; functional partitioning; sharding (partitioning key, fixed vs dynamic allocation, cross-shard query, global unique ID); Universal Scalability Law (serialization + crosstalk).
- **High availability** *(HPM Ch.12)*: MTBF/MTTR; single point of failure; shared storage/DRBD; synchronous clustering; failover/failback; split-brain; fencing.
- **Backup & recovery** *(HPM Ch.15)*: logical vs raw backup; snapshot (LVM); hot backup (XtraBackup/pg_basebackup); PITR bằng binlog/WAL; RPO/RTO; **backup chưa test = chưa có backup**; replica ≠ backup.
- **Monitoring & diagnostics** *(HPM Ch.3, Ch.16)*: status counters, process list, `SHOW ENGINE INNODB STATUS`, Performance Schema, `pg_stat_*`; trigger + thu thập dữ liệu khi sự cố; phân biệt work vs wait.
- **Hardware & OS** *(HPM Ch.9)*: CPU nhanh vs nhiều core, RAM/working set, HDD vs SSD vs PCIe flash, RAID level, write-back cache + BBU, filesystem (XFS/ext4), I/O scheduler, swappiness, đừng để swap.
- **Cloud** *(HPM Ch.13)*: IaaS vs DBaaS (RDS), I/O biến động, working set phải vừa RAM, hạn chế truy cập hệ thống.
- **Benchmarking** *(HPM Ch.2)*: sysbench, tpcc, chạy đủ lâu đến steady state, warmup, đo percentile, vẽ đồ thị, không kết luận từ 1 lần chạy 60 giây.
- **Security & safety settings** *(HPM Ch.8)*: `skip_name_resolve`, `read_only` cho replica, `sql_mode`, privilege tối thiểu.

## Tầng 3 — Interview & System Reasoning

- Giải thích một transaction từ `BEGIN` đến `COMMIT`: lock, undo, redo, binlog, fsync, MVCC visibility.
- Debug: "tăng đột biến `Threads_running`", "replication lag tăng", "deadlock xuất hiện sau release", "disk đầy", "query nhanh trên dev nhưng chậm trên prod".
- Thiết kế: chọn isolation level, chiến lược sharding, thiết kế HA/backup cho một hệ thống cụ thể; nêu RPO/RTO.
- Phân tích trade-off: durability vs throughput (`sync_binlog`, `innodb_flush_log_at_trx_commit`, `synchronous_commit`), consistency vs availability.

### Bài tập thực hành
1. Mở hai session, tái hiện dirty read / non-repeatable read / phantom ở từng isolation level trên MySQL và PostgreSQL; so sánh kết quả.
2. Tái hiện deadlock bằng hai transaction cập nhật hai dòng theo thứ tự ngược; đọc `SHOW ENGINE INNODB STATUS` / log PostgreSQL.
3. Quan sát gap lock/next-key lock: `SELECT ... FOR UPDATE` trên range ở REPEATABLE READ rồi chèn từ session khác.
4. Kill `mysqld`/`postgres` giữa chừng transaction ghi nhiều; quan sát crash recovery qua log.
5. Dựng replication MySQL (1 master + 1 replica) và PostgreSQL streaming replication; đo lag dưới tải ghi.
6. Thực hiện PITR: backup + binlog/WAL, "vô tình" `DROP TABLE`, khôi phục đến trước thời điểm đó.
7. Chạy sysbench OLTP với nhiều mức concurrency; vẽ throughput/latency theo số thread.

---

# PHẦN VI — MYSQL & INNODB

## Tầng 1 — Core Internals

### 1. Kiến trúc & storage engine *(HPM Ch.1)*
- Ba lớp: connection handling → query processing (parser/optimizer/cache) → storage engine API.
- InnoDB vs MyISAM vs Memory vs Archive vs NDB: transaction, row lock, crash safety, clustered index, hot backup.
- Chọn engine theo workload: logging, read-mostly, order processing, large volumes; lý do "dùng InnoDB trừ khi có lý do khác".
- Lịch sử/phiên bản: InnoDB plugin, MySQL 5.5 mặc định InnoDB, các fork (Percona Server, MariaDB, Drizzle) *(HPM Appendix A)*.

### 2. InnoDB internals *(HPM Ch.1, Ch.5, Ch.8, Appendix B)*
- Clustered index, secondary index, adaptive hash index, insert/change buffer, doublewrite buffer.
- Buffer pool (size, instances, young/old, dirty page flush), log buffer, redo log size & checkpoint, `innodb_flush_method`, `innodb_file_per_table`.
- MVCC & undo: history list length, purge, long-running transaction làm phình undo; `innodb_max_purge_lag`.
- Locking: next-key lock chặn phantom ở REPEATABLE READ; two-phase locking; lock `SELECT ... FOR UPDATE` yêu cầu truy cập PK (không dùng covering).
- Cách đọc `SHOW ENGINE INNODB STATUS`: SEMAPHORES, LATEST DETECTED DEADLOCK, TRANSACTIONS, FILE I/O, INSERT BUFFER, LOG, BUFFER POOL AND MEMORY.

### 3. MyISAM & các engine khác
- Table lock, concurrent insert, không crash-safe, key cache, delayed key write; vì sao không dùng mặc định.
- MyISAM `COUNT(*)` nhanh chỉ cho trường hợp không có `WHERE`.

### 4. Query cache & prepared statements *(HPM Ch.7)*
- Query cache: cơ chế, invalidation thô, mutex toàn cục, vì sao thường tắt (MySQL 8 đã gỡ).
- Prepared statement 3 loại: client-side emulated, server-side (binary protocol), SQL interface; lợi ích và giới hạn.

### 5. Partitioning, view, stored code *(HPM Ch.7)*
- Partitioned table: range/hash/list/key; pruning; điều kiện để prune; NULL làm lệch pruning; chi phí open/lock mọi partition.
- View: MERGE vs TEMPTABLE; giới hạn; không có materialized view native.
- Trigger, stored procedure, event, cursor: ưu/nhược, tác động tới replication, khó profile.

## Tầng 2 — Production MySQL

### 1. Cấu hình *(HPM Ch.8)*
- Cách đọc cấu hình: scope (global/session), dynamic variable, side effect khi `SET`.
- Cấu hình tối thiểu: `innodb_buffer_pool_size`, `innodb_log_file_size`, `innodb_flush_method`, `max_connections`, `thread_cache_size`, `table_cache`, `tmp_table_size`.
- Tư tưởng: đừng "tuning by ratio", đừng tin tuning script, đừng chỉnh `sort_buffer_size` bừa; dùng mặc định trừ khi có bằng chứng.
- Safety/sanity: `sql_mode`, `expire_logs_days`, `skip_name_resolve`, `innodb=FORCE`, `read_only`, `skip_slave_start`.
- Cách đặt buffer pool: tổng RAM − OS − per-connection − log cache − cache khác, chừa dư.

### 2. Replication *(HPM Ch.10)*
- Cơ chế 3 bước: binlog → relay log → SQL thread; I/O thread, SQL thread.
- Setup replica, clone replica (snapshot/XtraBackup), `CHANGE MASTER TO`, log coordinates.
- Statement-based vs row-based: ưu nhược; nondeterministic statements, temporary tables, `UUID()`, `LIMIT` không `ORDER BY`.
- Topology: master–replicas, master–master active/passive, distribution master, ring (tránh), multisource emulation.
- Lag: đo bằng heartbeat thay vì `Seconds_behind_master`; vì sao replica single-threaded chậm; writes không scale bằng replication.
- Kiểm tra nhất quán (`pt-table-checksum`), sửa lệch (`pt-table-sync`), thay master, promote replica (planned/unplanned).
- Lỗi thường gặp: server_id trùng, replica ghi trực tiếp, filter (`binlog_do_db`) nguy hiểm, mixed engine, mất binlog.

### 3. Scaling, HA, backup *(HPM Ch.11, 12, 15)*
- Read/write splitting và vấn đề dữ liệu cũ (stale read): query-based, session-based, version-based.
- Sharding, global ID (`auto_increment_offset`, ID trong bit cao), directory server.
- HA: Pacemaker/DRBD, Percona XtraDB Cluster/Galera, NDB, replication manager (hạn chế của MMM/MHA).
- Backup: `mysqldump --single-transaction --master-data`, XtraBackup, LVM snapshot; restore time; PITR; test restore.

### 4. Phiên bản hiện đại (cần tự bổ sung từ manual)
- MySQL 8.x: window function, CTE, hash join, histogram, instant DDL, atomic DDL, invisible index, descending index, `EXPLAIN ANALYZE`, bỏ query cache, group replication, `utf8mb4` mặc định.

## Tầng 3 — Interview & Reasoning

- Vì sao InnoDB secondary index lớn khi PK lớn? Hệ quả khi chọn UUID làm PK?
- Giải thích REPEATABLE READ của InnoDB và vì sao khác chuẩn ANSI.
- Replica bị lag: quy trình điều tra và các cách giảm lag.
- Thiết kế quy trình backup/restore cho DB 2 TB với RTO 1 giờ, RPO 5 phút.

### Bài tập thực hành MySQL
1. So sánh insert 3 triệu dòng với PK `AUTO_INCREMENT` và PK UUID: thời gian, kích thước index (như bảng 5-1 trong sách).
2. Đọc `SHOW ENGINE INNODB STATUS` lúc có long transaction; quan sát history list length.
3. Dựng master–replica; tạo lag cố ý bằng query nặng; đo bằng heartbeat; so với `Seconds_behind_master`.
4. Backup bằng XtraBackup, khôi phục sang máy mới, gắn làm replica.
5. Thử nghiệm partition theo tháng cho bảng log; kiểm tra pruning bằng `EXPLAIN PARTITIONS`.
6. Thiết lập slow log, chạy workload mẫu, phân tích bằng `pt-query-digest`, tối ưu top-3 query.

---

# PHẦN VII — POSTGRESQL

## Tầng 1 — Core Internals

### 1. Kiến trúc
- Postmaster, backend process mỗi kết nối, shared buffers, WAL buffers, background workers (checkpointer, bgwriter, walwriter, autovacuum launcher/worker).
- Cluster, database, schema, tablespace; system catalog (`pg_class`, `pg_attribute`, `pg_stat_*`).
- Tại sao cần connection pooler (PgBouncer): chi phí process, `max_connections`.

### 2. Storage
- Heap file, page 8 KB, tuple header (`xmin`, `xmax`, `ctid`), line pointer, free space map, visibility map.
- TOAST: lưu giá trị lớn; compression; ảnh hưởng tới `SELECT *`.
- Fill factor, HOT update (heap-only tuple); bloat.

### 3. MVCC & VACUUM
- Mỗi UPDATE tạo tuple version mới; dead tuple; snapshot và visibility rule; transaction ID (`xid`), wraparound và freeze.
- `VACUUM`, `VACUUM FULL`, `autovacuum` (threshold, cost-based delay), `ANALYZE`; bloat bảng và index; long transaction chặn vacuum.
- So sánh với InnoDB: undo log tách riêng vs version trong heap; hệ quả với UPDATE-heavy workload.

### 4. Index & planner
- Loại index: B-Tree, Hash, GIN (full-text, array, jsonb), GiST, SP-GiST, BRIN; partial index, expression index, covering index (`INCLUDE`), multicolumn.
- Index-only scan phụ thuộc visibility map; bitmap index scan kết hợp nhiều index.
- Planner: statistics (`pg_stats`, `n_distinct`, MCV, histogram), `default_statistics_target`, extended statistics; cost params (`random_page_cost`, `effective_cache_size`, `work_mem`).
- Join: nested loop, hash join, merge join; `enable_*` chỉ để chẩn đoán; CTE materialization (PG12+ inlining).

### 5. WAL, checkpoint, replication
- WAL, LSN, `wal_level`, `fsync`, `synchronous_commit`, `full_page_writes`, checkpoint (`checkpoint_timeout`, `max_wal_size`).
- Streaming replication (physical), replication slot, hot standby, logical replication/publication-subscription; PITR (base backup + WAL archive); `pg_basebackup`, `pg_dump`.

### 6. Tính năng riêng
- Isolation: READ COMMITTED mặc định, REPEATABLE READ (snapshot), SERIALIZABLE (SSI, serialization failure → retry).
- Locking: row-level lock mode, advisory lock, `FOR UPDATE SKIP LOCKED / NOWAIT`.
- JSONB, array, range type, `LISTEN/NOTIFY`, partitioning declarative, extensions, `RETURNING`, `ON CONFLICT`.

## Tầng 2 — Production PostgreSQL

- Cấu hình cơ bản: `shared_buffers`, `work_mem`, `maintenance_work_mem`, `effective_cache_size`, `max_wal_size`, `autovacuum_*`.
- Giám sát: `pg_stat_activity`, `pg_stat_statements`, `pg_locks`, `pg_stat_user_tables` (dead tuples), `pg_stat_replication`; phát hiện bloat, long transaction, idle in transaction.
- Xử lý query chậm: `EXPLAIN (ANALYZE, BUFFERS)`, so sánh estimate vs actual, `ANALYZE` lại, thêm index (concurrently), partial/expression index.
- `CREATE INDEX CONCURRENTLY`, migration không khóa dài, `lock_timeout`.
- Backup/restore, PITR, failover (Patroni mức nhận biết), logical replication cho upgrade.
- Vấn đề phổ biến: transaction ID wraparound, autovacuum không kịp, connection storm, bloat index, kế hoạch thay đổi sau `ANALYZE`.

## Tầng 3 — Interview & Reasoning

- So sánh MVCC PostgreSQL và InnoDB: hệ quả về UPDATE, vacuum/purge, storage, index maintenance.
- Giải thích "vì sao bảng không hết dữ liệu mà vẫn lớn dần" (bloat) và cách xử lý.
- Khi nào chọn PostgreSQL thay MySQL và ngược lại.

### Bài tập thực hành PostgreSQL
1. Quan sát `xmin/xmax/ctid` qua UPDATE/DELETE; chạy `VACUUM` và xem thay đổi.
2. Tạo bloat có chủ đích (UPDATE hàng loạt + long transaction); đo kích thước bảng trước/sau `VACUUM`, `VACUUM FULL`.
3. So sánh Seq Scan, Index Scan, Bitmap Scan, Index Only Scan với selectivity khác nhau; ghi ngưỡng chuyển plan.
4. Tạo partial index, expression index, GIN trên `jsonb`; kiểm chứng bằng `EXPLAIN ANALYZE`.
5. Dựng streaming replication và PITR; thử failover thủ công.
6. Tái hiện serialization failure ở SERIALIZABLE và viết đoạn retry.

---

# PHẦN VIII — SPRING DATA JPA & HIBERNATE

## Tầng 1 — Core Internals

### 1. JPA/Hibernate nền tảng
- JPA (specification) vs Hibernate (implementation) vs Spring Data JPA (abstraction); các lớp: `EntityManager`, `Session`, `SessionFactory`, `JpaRepository`.
- Mapping: `@Entity`, `@Id`, `@GeneratedValue` (IDENTITY, SEQUENCE, UUID; ảnh hưởng tới batch insert), `@Column`, `@Embeddable`, `@Enumerated`, `@Version`.
- Quan hệ: `@OneToMany`, `@ManyToOne`, `@ManyToMany`, `@OneToOne`; owning side vs inverse side (`mappedBy`); đồng bộ hai chiều.
- Inheritance mapping: SINGLE_TABLE, JOINED, TABLE_PER_CLASS; trade-off schema/performance.

### 2. Persistence context & entity lifecycle
- Trạng thái: transient, managed (persistent), detached, removed; `persist/merge/remove/refresh/detach`.
- Persistence context = first-level cache + identity map; dirty checking; **flush** (khi nào xảy ra: commit, trước query, `flush()`), flush mode.
- Write-behind: SQL không chạy ngay lúc gọi `save()`; thứ tự SQL (insert/update/delete ordering).
- Second-level cache và query cache (khái niệm, rủi ro invalidation) — liên hệ với bài học caching ở HPM Ch.14.
- `equals/hashCode` cho entity (business key vs id), proxy và `getClass`.

### 3. Fetching & lazy loading
- `FetchType.LAZY` vs `EAGER`; proxy/bytecode enhancement; `LazyInitializationException` và `open-in-view`.
- **N+1 select problem**: nguyên nhân, cách phát hiện (log SQL, statistics), cách xử lý (`JOIN FETCH`, `@EntityGraph`, `@BatchSize`, subselect, DTO projection).
- `JOIN FETCH` với collection + phân trang (`HHH000104`: pagination in memory), `MultipleBagFetchException`, cartesian product.
- Projection: interface/class-based DTO, `@Query` với constructor expression, native query.

### 4. Query & Spring Data
- Derived query method, `@Query` (JPQL/native), `Specification`, Criteria API, Querydsl (nhận biết).
- `Pageable/Page/Slice`: `Page` chạy thêm `COUNT` (liên hệ `SQL_CALC_FOUND_ROWS` ở HPM Ch.6); keyset pagination thay cho offset.
- Bulk update/delete (`@Modifying`) bỏ qua persistence context → cần clear/flush.
- Auditing (`@CreatedDate`, `@LastModifiedBy`), soft delete (`@SQLRestriction`/`@Where`).

### 5. Transaction trong Spring
- `@Transactional`: proxy, propagation (REQUIRED, REQUIRES_NEW, ...), isolation, `readOnly`, timeout, rollback rules (unchecked mặc định).
- Self-invocation không đi qua proxy; `@Transactional` trên method private/final; transaction manager (`JpaTransactionManager`).
- Giữ transaction ngắn; không gọi HTTP/IO chậm trong transaction (kéo dài lock, giữ connection — liên hệ HPM Ch.1, Ch.7).
- `TransactionSynchronization`, `@TransactionalEventListener` (AFTER_COMMIT), outbox pattern ở mức khái niệm.

### 6. Locking & concurrency ở tầng ORM
- Optimistic locking với `@Version` (và `OptimisticLockException`); pessimistic: `LockModeType.PESSIMISTIC_WRITE/READ` ↔ `SELECT ... FOR UPDATE`/`FOR SHARE`.
- Lost update, write skew và cách phòng; retry khi optimistic fail; `SKIP LOCKED` cho queue.
- Deadlock từ thứ tự cập nhật nhiều entity; sắp xếp khóa nhất quán.

## Tầng 2 — Production JPA/Hibernate

- **Bật quan sát SQL**: `hibernate.show_sql`/logger `org.hibernate.SQL`, `hibernate.generate_statistics`, datasource-proxy/p6spy, slow query log DB; đếm số query mỗi request.
- **Batch**: `hibernate.jdbc.batch_size`, `order_inserts/order_updates`, IDENTITY làm vô hiệu hóa batch insert → dùng SEQUENCE; `saveAll` với chunk + `flush/clear`.
- **Connection pool (HikariCP)**: kích thước pool theo concurrency thực tế (liên hệ "working concurrency" HPM Ch.2, Ch.14); `maximumPoolSize`, `connectionTimeout`; tránh giữ connection khi chờ IO ngoài.
- **Schema & migration**: Flyway/Liquibase; không dùng `ddl-auto=update` ở production; đồng bộ index trong migration; ORM sinh schema kém (HPM Ch.4 "autogenerated schemas").
- **Index cho JPA query**: kiểm tra SQL thực tế sinh ra, `EXPLAIN` từng query; index phù hợp với `ORDER BY`/`WHERE` của repository method.
- **Pagination/Sort lớn**: tránh offset sâu; keyset; read replica routing (`AbstractRoutingDataSource`) và vấn đề stale read (HPM Ch.11).
- **Caching**: second-level cache (Ehcache/Redis) — khi nào đáng; cache invalidation; stampede.
- **Transaction trong hệ thống nhiều instance**: optimistic/pessimistic lock, idempotency, retry; không dùng `synchronized` JVM để thay lock DB.
- **Testing**: Testcontainers với đúng DB (MySQL/PostgreSQL) thay vì H2; `@DataJpaTest`; test N+1 bằng đếm query; test rollback/propagation.
- **Version awareness**: Hibernate 6 thay đổi HQL/ID generator/`@Type`; Spring Boot 3 dùng Jakarta namespace — kiểm tra theo phiên bản thực tế.

## Tầng 3 — Interview & Reasoning

- Giải thích N+1 bằng ví dụ, phát hiện và sửa; so sánh các cách sửa (`JOIN FETCH` vs `@EntityGraph` vs batch fetch vs DTO).
- Vì sao `save()` không thấy SQL ngay? Khi nào Hibernate flush? Hậu quả dirty checking với entity lớn.
- `@Transactional` không hoạt động trong tình huống nào? Làm sao viết test chứng minh?
- Chọn optimistic hay pessimistic locking cho: giảm tồn kho, đăng ký tín chỉ, nhận job từ queue.
- Khi nào nên bỏ ORM và dùng SQL/JdbcTemplate/jOOQ cho một use case?

### Bài tập thực hành JPA/Hibernate
1. Tạo mô hình `Order–OrderItem–Product`; tái hiện N+1; sửa bằng 4 cách; đếm query và so sánh.
2. Tái hiện `LazyInitializationException`; sửa không dùng `EAGER`.
3. So sánh `GenerationType.IDENTITY` và `SEQUENCE` khi insert 100k dòng với batch bật/tắt.
4. Tái hiện lost update; sửa bằng `@Version` và bằng `PESSIMISTIC_WRITE`; viết test đa luồng.
5. Viết `@Transactional` self-invocation bug + test; sửa bằng tách bean.
6. Viết API phân trang bằng `Pageable` và bằng keyset; so sánh `EXPLAIN` và thời gian trên bảng lớn.
7. Dựng integration test bằng Testcontainers (PostgreSQL và MySQL) kiểm tra transaction rollback và deadlock retry.
8. Bật `generate_statistics`, đo số statement, cache hit, flush count cho một use case thật.

---

# PHẦN IX — BÀI TẬP CỐT LÕI VÀ KẾ HOẠCH ÔN TẬP

## Bài tập cốt lõi

1. Duy trì ngân hàng 100–120 câu hỏi, phân nhóm rõ (SQL, Index/Optimization, Internals, MySQL, PostgreSQL, JPA). Tài liệu này cung cấp 120 câu khởi đầu.
2. Mỗi ngày trả lời 5–10 câu **không nhìn tài liệu**; nói thành tiếng hoặc viết theo khung năm bước.
3. Mỗi tuần một buổi mock interview; câu hỏi ngẫu nhiên; giới hạn thời gian.
4. Ghi lại câu trả lời sai/thiếu cơ chế/thiếu limitation; viết lại bằng lời của mình.
5. Với câu hỏi cơ chế, **luôn tạo thí nghiệm nhỏ** (EXPLAIN, hai session, benchmark) thay vì chỉ đọc.
6. Mỗi 2 tuần làm lại các câu từng sai.
7. Đánh giá tiến độ bằng khả năng giải thích, đo đạc và debug, không bằng số trang đã đọc.

## Lịch luyện mẫu trong một tuần

| Ngày | Việc cần làm | Sản phẩm đầu ra |
|---|---|---|
| Thứ 2 | Học một chủ đề mới; đọc manual/sách | Ghi chú khái niệm, thuật ngữ |
| Thứ 3 | Viết SQL/code tối thiểu, chạy EXPLAIN | Script + output |
| Thứ 4 | Thử edge case, lỗi, giới hạn | Danh sách failure mode |
| Thứ 5 | Trả lời 5–10 câu không nhìn tài liệu | Bản trả lời nháp |
| Thứ 6 | Ôn câu sai; đọc sâu phần chưa hiểu | Bản sửa + câu hỏi cần làm rõ |
| Thứ 7 | Mock interview 30–60 phút | Điểm mạnh, lỗ hổng, kế hoạch sửa |
| Chủ nhật | Ôn nhẹ hoặc nghỉ; lên kế hoạch tuần sau | Danh sách ưu tiên |

## Rubric chấm câu trả lời

| Điểm | Mô tả |
|---|---|
| 0 | Không biết hoặc sai bản chất |
| 1 | Nhớ định nghĩa nhưng không giải thích được cơ chế |
| 2 | Giải thích đúng cơ chế nhưng thiếu ví dụ/số đo hoặc limitation |
| 3 | Có cơ chế + ví dụ chính xác (EXPLAIN/SQL) + nêu được ít nhất một giới hạn |
| 4 | So sánh được trade-off giữa các phương án/DBMS, nêu giả định, áp dụng vào tình huống thực tế |

Mục tiêu trước phỏng vấn: phần lớn câu trọng tâm đạt mức 3; các chủ đề cốt lõi của vị trí (SQL, index, transaction, JPA) đạt mức 4.

## Dự án tích hợp gợi ý (portfolio)

- Hệ thống đăng ký tín chỉ/POS: Spring Boot + JPA + PostgreSQL (hoặc MySQL), có Flyway, Testcontainers, đo query, tối ưu index, xử lý đồng thời (optimistic/pessimistic), báo cáo benchmark trước/sau tối ưu.
- Viết một "báo cáo tối ưu" cho dự án: profile → phát hiện top query → giải pháp → số đo → trade-off.

---

# PHẦN X — NGÂN HÀNG 120 CÂU HỎI PHỎNG VẤN

Chỉ có đề bài, chưa có đáp án. Hãy ghi câu trả lời riêng theo khung năm bước.

## A. SQL & Schema Design

**Q01.** Thứ tự thực thi logic của một câu `SELECT` là gì? Vì sao không dùng được alias của `SELECT` trong `WHERE`?
**Q02.** `NULL` hoạt động thế nào trong so sánh, `JOIN`, `GROUP BY`, `COUNT(col)` và `COUNT(*)`?
**Q03.** `INNER JOIN`, `LEFT JOIN`, `FULL JOIN` khác nhau thế nào? Đặt điều kiện ở `ON` và ở `WHERE` của `LEFT JOIN` khác gì?
**Q04.** `IN`, `EXISTS`, `JOIN` và `NOT IN` có thể trả kết quả khác nhau khi nào (đặc biệt với `NULL`)?
**Q05.** Correlated subquery là gì? MySQL cũ xử lý thế nào và nên viết lại ra sao?
**Q06.** `UNION` khác `UNION ALL` thế nào về kết quả và chi phí?
**Q07.** `WHERE` khác `HAVING` thế nào? Khi nào `GROUP BY` cột không nằm trong `SELECT` gây vấn đề?
**Q08.** Window function là gì? So sánh `ROW_NUMBER`, `RANK`, `DENSE_RANK`; viết top-N per group.
**Q09.** CTE khác subquery/derived table/view thế nào? Khi nào CTE bị materialize?
**Q10.** Viết truy vấn tìm bản ghi trùng và xóa trùng giữ lại một bản; nêu rủi ro khi chạy trên bảng lớn.
**Q11.** `DECIMAL` khác `FLOAT/DOUBLE` thế nào? Lưu tiền nên dùng gì và vì sao?
**Q12.** `CHAR` khác `VARCHAR` khác `TEXT` thế nào về lưu trữ, so sánh trailing space, temp table?
**Q13.** `DATETIME` khác `TIMESTAMP` thế nào? Lưu thời gian trong hệ thống đa múi giờ nên làm gì?
**Q14.** Chọn khóa chính: auto-increment, UUID, natural key — ưu/nhược và ảnh hưởng tới index/insert?
**Q15.** Chuẩn hóa là gì (1NF–BCNF)? Khi nào phi chuẩn hóa có chủ đích là hợp lý?
**Q16.** Summary table, cache table, materialized view: khác nhau thế nào và cập nhật ra sao?
**Q17.** Foreign key giúp gì và tốn gì? Khi nào bỏ foreign key ở production?
**Q18.** EAV (entity-attribute-value) vì sao là anti-pattern? Thay thế bằng gì?
**Q19.** Thiết kế queue table trong RDBMS: vì sao tránh `SELECT ... FOR UPDATE` kiểu cũ, claim row thế nào?
**Q20.** Thay đổi schema trên bảng lớn không gây downtime: có những kỹ thuật nào?

## B. Indexing & Query Optimization

**Q21.** B-Tree index lưu và tìm dữ liệu thế nào? Vì sao hỗ trợ range scan và `ORDER BY`?
**Q22.** Quy tắc leftmost prefix là gì? Với index `(a,b,c)`, query nào dùng được, query nào không?
**Q23.** Vì sao điều kiện range làm các cột sau nó trong composite index không được dùng để lọc?
**Q24.** Selectivity và cardinality là gì? Chọn thứ tự cột composite index dựa trên gì (và quy tắc "most selective first" sai khi nào)?
**Q25.** Covering index là gì? Làm sao nhận ra qua `EXPLAIN`? Ảnh hưởng của InnoDB secondary index chứa PK?
**Q26.** Prefix index hoạt động thế nào? Chọn độ dài prefix và hạn chế (không dùng cho ORDER BY/covering)?
**Q27.** Vì sao `WHERE DATE(created_at) = ...` hoặc `WHERE id + 1 = 5` không dùng được index? Cách viết lại?
**Q28.** Implicit type conversion/collation mismatch làm mất index thế nào?
**Q29.** Index có thể làm chậm điều gì? Chi phí duy trì index khi INSERT/UPDATE/DELETE?
**Q30.** Redundant, duplicate và unused index: nhận diện và xử lý thế nào?
**Q31.** Hash index, full-text index, spatial index khác B-Tree ở đâu? Khi nào dùng?
**Q32.** Đọc `EXPLAIN`: ý nghĩa của `type`, `key`, `key_len`, `rows`, `filtered`, `Extra`.
**Q33.** `Using filesort` và `Using temporary` nghĩa là gì? Khi nào chúng là vấn đề thực sự?
**Q34.** `EXPLAIN` khác `EXPLAIN ANALYZE` thế nào? Vì sao estimate có thể sai?
**Q35.** Optimizer chọn join order thế nào? Khi nào dùng `STRAIGHT_JOIN`/hint và rủi ro?
**Q36.** `LIMIT 100000, 20` chậm vì sao? So sánh deferred join và keyset pagination.
**Q37.** `COUNT(*)` chậm trên bảng lớn InnoDB vì sao? Các cách tối ưu/xấp xỉ?
**Q38.** `SQL_CALC_FOUND_ROWS` và `Page` của Spring Data có chi phí gì? Thay bằng gì?
**Q39.** Thống kê (statistics) của optimizer lấy từ đâu? Khi nào cần `ANALYZE`?
**Q40.** Quy trình xử lý một query chậm từ slow log đến lúc triển khai index (profiling → EXPLAIN → thay đổi → đo lại)?

## C. Database Internals

**Q41.** Mô tả đường đi của một query từ client đến storage engine rồi trở về.
**Q42.** WAL là gì? Vì sao ghi log tuần tự trước giúp vừa nhanh vừa bền?
**Q43.** Redo log và undo log khác nhau thế nào về mục đích và thời điểm dùng?
**Q44.** Checkpoint là gì? Điều gì xảy ra nếu log quá nhỏ ("furious flushing")?
**Q45.** Crash recovery hoạt động thế nào (redo, undo)? Vì sao transaction đã commit không mất?
**Q46.** `fsync` và write-back cache của RAID/đĩa ảnh hưởng durability thế nào? Vì sao cần BBU?
**Q47.** Buffer pool/shared buffers làm gì? Cơ chế LRU, dirty page, read-ahead?
**Q48.** Doublewrite buffer / full-page writes giải quyết vấn đề gì?
**Q49.** ACID: thành phần nào của DBMS bảo đảm từng chữ cái?
**Q50.** Bốn isolation level chuẩn và các hiện tượng dirty/non-repeatable/phantom; so sánh cách MySQL và PostgreSQL triển khai.
**Q51.** MVCC là gì? Một transaction thấy phiên bản nào của một dòng (read view/snapshot)?
**Q52.** Phantom read xảy ra thế nào và InnoDB ngăn bằng next-key lock ra sao?
**Q53.** Lost update và write skew là gì? Phòng tránh thế nào?
**Q54.** Shared lock, exclusive lock, intention lock, gap lock, next-key lock khác nhau thế nào?
**Q55.** Deadlock hình thành thế nào? Hệ thống phát hiện và xử lý ra sao? Giảm deadlock bằng cách nào?
**Q56.** Optimistic và pessimistic concurrency control: khác nhau, khi nào chọn cái nào?
**Q57.** Row-based, statement-based, mixed replication khác nhau thế nào? Rủi ro của từng loại?
**Q58.** Vì sao replication bất đồng bộ có thể mất dữ liệu khi master hỏng? Semisynchronous và synchronous giải quyết gì?
**Q59.** B-Tree và LSM-tree khác nhau thế nào? Vì sao RDBMS OLTP thường dùng B-Tree?
**Q60.** Page split, fragmentation và fill factor ảnh hưởng hiệu năng thế nào?

## D. MySQL & InnoDB

**Q61.** Kiến trúc MySQL tách server layer và storage engine; điều này mang lại và hạn chế gì?
**Q62.** InnoDB khác MyISAM ở những điểm nào? Khi nào (nếu có) vẫn dùng MyISAM?
**Q63.** Clustered index trong InnoDB là gì? Hệ quả với secondary index và với việc chọn PK?
**Q64.** Vì sao chèn PK ngẫu nhiên (UUID) chậm hơn PK tuần tự trong InnoDB?
**Q65.** Secondary index lookup trong InnoDB cần mấy lần tra cứu? Adaptive hash index giúp gì?
**Q66.** Insert buffer/change buffer là gì? Khi nào hữu ích?
**Q67.** REPEATABLE READ của InnoDB khác chuẩn ANSI ở đâu? Vì sao mặc định dùng nó?
**Q68.** `SELECT ... FOR UPDATE` khóa những gì? Vì sao có thể khóa nhiều hơn dự kiến (không có index)?
**Q69.** Long-running transaction ảnh hưởng undo log/history list thế nào? Cách điều tra?
**Q70.** `innodb_flush_log_at_trx_commit` (0/1/2) và `sync_binlog` đánh đổi gì?
**Q71.** `innodb_buffer_pool_size` và `innodb_log_file_size` nên đặt dựa trên gì? Vì sao "75% RAM" là quy tắc không đáng tin?
**Q72.** Query cache hoạt động thế nào và vì sao thường nên tắt (MySQL 5.x) / đã bị gỡ (MySQL 8)?
**Q73.** Prepared statement: ba loại (client-side, server-side, SQL interface) khác nhau thế nào?
**Q74.** Partitioned table giúp gì và có những cạm bẫy nào (pruning, NULL, mismatch index, chi phí mở partition)?
**Q75.** MySQL replication gồm những thread nào? Mô tả luồng từ binlog đến dữ liệu trên replica.
**Q76.** Vì sao replica hay bị lag? Cách đo lag đúng và cách giảm lag?
**Q77.** Làm sao kiểm tra và sửa dữ liệu lệch giữa master và replica?
**Q78.** Quy trình promote replica thành master khi master hỏng (planned vs unplanned) và rủi ro mất dữ liệu?
**Q79.** Trình bày chiến lược backup MySQL: logical vs raw, `mysqldump --single-transaction`, XtraBackup, snapshot; thế nào là PITR?
**Q80.** Vì sao "replica không phải backup"? Ba bước test backup/restore tối thiểu?

## E. PostgreSQL

**Q81.** Kiến trúc process của PostgreSQL và vì sao cần connection pooler?
**Q82.** PostgreSQL lưu tuple và MVCC thế nào (`xmin`, `xmax`, `ctid`)? Khác InnoDB ở điểm nào?
**Q83.** Dead tuple và bloat hình thành thế nào? Hậu quả với bảng UPDATE nhiều?
**Q84.** `VACUUM`, `VACUUM FULL`, `ANALYZE` và autovacuum làm gì? Điều gì khiến autovacuum không theo kịp?
**Q85.** Transaction ID wraparound là gì? Vì sao nguy hiểm và phòng tránh thế nào?
**Q86.** HOT update và fill factor có tác dụng gì?
**Q87.** TOAST là gì và ảnh hưởng gì tới `SELECT *`?
**Q88.** Các loại index trong PostgreSQL (B-Tree, Hash, GIN, GiST, BRIN): dùng khi nào?
**Q89.** Partial index, expression index, covering index (`INCLUDE`): ví dụ và lợi ích?
**Q90.** Index-only scan phụ thuộc visibility map thế nào? Vì sao đôi khi vẫn đọc heap?
**Q91.** Seq Scan, Index Scan, Bitmap Heap Scan: khi nào planner chọn cái nào?
**Q92.** Cách đọc `EXPLAIN (ANALYZE, BUFFERS)`: estimated vs actual rows, loops, shared hit/read?
**Q93.** Planner cost parameter (`random_page_cost`, `effective_cache_size`, `work_mem`) ảnh hưởng plan thế nào?
**Q94.** Nested loop, hash join, merge join: điều kiện phù hợp của từng loại?
**Q95.** Isolation level trong PostgreSQL: READ COMMITTED, REPEATABLE READ, SERIALIZABLE (SSI) khác MySQL ra sao?
**Q96.** `SELECT ... FOR UPDATE SKIP LOCKED/NOWAIT` và advisory lock dùng khi nào?
**Q97.** WAL, checkpoint, `synchronous_commit` và `full_page_writes` ảnh hưởng durability/hiệu năng thế nào?
**Q98.** Streaming replication và logical replication khác nhau ra sao? Replication slot có rủi ro gì?
**Q99.** `CREATE INDEX CONCURRENTLY` làm gì và hạn chế? Làm migration an toàn với `lock_timeout` thế nào?
**Q100.** So sánh chọn PostgreSQL và MySQL cho một hệ thống OLTP cụ thể; nêu tiêu chí.

## F. Spring Data JPA & Hibernate

**Q101.** JPA, Hibernate và Spring Data JPA khác nhau thế nào?
**Q102.** Persistence context là gì? Các trạng thái entity (transient, managed, detached, removed)?
**Q103.** Dirty checking hoạt động thế nào và khi nào xảy ra flush?
**Q104.** Vì sao `repository.save()` không chạy SQL ngay? Thứ tự SQL do Hibernate sinh ra?
**Q105.** `LAZY` và `EAGER` khác nhau thế nào? `LazyInitializationException` nguyên nhân và cách sửa đúng?
**Q106.** N+1 select là gì? Phát hiện và sửa bằng những cách nào; so sánh các cách?
**Q107.** `JOIN FETCH` với collection và phân trang có vấn đề gì? `MultipleBagFetchException`?
**Q108.** DTO projection khác trả entity ở điểm nào? Khi nào nên dùng?
**Q109.** `GenerationType.IDENTITY` vs `SEQUENCE` ảnh hưởng thế nào tới batch insert?
**Q110.** Cấu hình JDBC batch (`batch_size`, `order_inserts`) và thực hành insert 100k dòng đúng cách?
**Q111.** `@Transactional`: propagation, isolation, `readOnly`, rollback rules; mặc định rollback với exception nào?
**Q112.** Vì sao `@Transactional` không hoạt động khi self-invocation hoặc trên method private? Cách kiểm chứng?
**Q113.** Giữ transaction/connection quá lâu (gọi HTTP bên trong transaction) gây vấn đề gì?
**Q114.** Optimistic locking với `@Version` hoạt động thế nào? Xử lý `OptimisticLockException`?
**Q115.** Pessimistic locking trong JPA (`PESSIMISTIC_WRITE`) ánh xạ thành SQL gì? Rủi ro deadlock?
**Q116.** `@Modifying` bulk update bỏ qua persistence context thế nào và xử lý ra sao?
**Q117.** `Page` vs `Slice` vs keyset pagination trong Spring Data; chi phí `COUNT`?
**Q118.** First-level cache khác second-level cache thế nào? Khi nào bật second-level cache?
**Q119.** Vì sao không dùng `ddl-auto=update` ở production? Quản lý schema bằng Flyway/Liquibase thế nào?
**Q120.** Làm sao phát hiện và chứng minh vấn đề hiệu năng JPA (đếm query, log SQL, `generate_statistics`, `EXPLAIN` từng query)? Khi nào chuyển sang SQL thuần?

---

# PHẦN XI — CHECKLIST TỰ ĐÁNH GIÁ

## SQL & Schema
- □ Tôi viết đúng JOIN/subquery/window function và dự đoán được kết quả khi có `NULL`.
- □ Tôi chọn đúng data type (tiền, thời gian, định danh, chuỗi) và giải thích được lý do.
- □ Tôi thiết kế schema có chuẩn hóa hợp lý và biết khi nào phi chuẩn hóa.
- □ Tôi biết cách thay đổi schema bảng lớn không gây downtime.

## Index & Query Optimization
- □ Tôi giải thích được B-Tree, leftmost prefix, selectivity, covering index.
- □ Tôi đọc được `EXPLAIN`/`EXPLAIN ANALYZE` và chỉ ra điểm nghẽn.
- □ Tôi biết các nguyên nhân khiến index không được dùng và cách sửa.
- □ Tôi tối ưu được pagination, `COUNT`, `GROUP BY`, subquery lớn.
- □ Tôi dùng slow log/profile để chọn query đáng tối ưu thay vì đoán.

## Database Internals
- □ Tôi mô tả được WAL, redo/undo, checkpoint, crash recovery.
- □ Tôi giải thích được ACID và isolation levels, MVCC, các loại lock, deadlock.
- □ Tôi phân biệt được cách triển khai MVCC của InnoDB và PostgreSQL.
- □ Tôi hiểu buffer pool, page, fsync, durability trade-off.

## MySQL
- □ Tôi giải thích được clustered/secondary index của InnoDB và hệ quả với PK.
- □ Tôi biết các cấu hình cốt lõi và vì sao không "tuning by ratio".
- □ Tôi dựng, giám sát và sửa lỗi replication; biết tạo/khôi phục backup và PITR.

## PostgreSQL
- □ Tôi hiểu tuple versioning, VACUUM/autovacuum, bloat, XID wraparound.
- □ Tôi chọn đúng loại index và đọc được planner output.
- □ Tôi biết WAL, streaming/logical replication, PITR cơ bản.

## Spring Data JPA/Hibernate
- □ Tôi giải thích được persistence context, flush, dirty checking.
- □ Tôi phát hiện và sửa được N+1; biết cấu hình batch.
- □ Tôi viết đúng `@Transactional`, hiểu proxy/self-invocation, và chọn đúng locking strategy.
- □ Tôi kiểm thử với DB thật (Testcontainers) và quan sát SQL sinh ra.

## Interview readiness
- □ Tôi trả lời được 5–10 câu mỗi ngày mà không nhìn tài liệu.
- □ Tôi thực hiện ít nhất một mock interview mỗi tuần.
- □ Tôi ghi lại câu sai/thiếu cơ chế và viết lại.
- □ Tôi có thí nghiệm/số đo cho các khái niệm trọng tâm.
- □ Tôi nêu được limitation và trade-off, không chỉ định nghĩa.
- □ Tôi kiểm tra phiên bản MySQL/PostgreSQL/Hibernate/Spring trước khi khẳng định hành vi.

## Kế hoạch ôn lại cá nhân

| Chủ đề | Điểm hiện tại (0–4) | Lỗ hổng cần sửa | Ngày ôn lại |
|---|---|---|---|
| SQL — JOIN/Subquery/Window | | | |
| Schema design & data types | | | |
| Indexing | | | |
| Query optimization & EXPLAIN | | | |
| Transaction & isolation | | | |
| MVCC & locking | | | |
| WAL / recovery / buffer pool | | | |
| MySQL/InnoDB | | | |
| Replication / HA / backup | | | |
| PostgreSQL internals | | | |
| JPA/Hibernate — mapping & fetching | | | |
| JPA/Hibernate — transaction & locking | | | |
| Interview reasoning | | | |

*Kết thúc tài liệu. Hãy cập nhật checklist theo tiến độ thực tế; ưu tiên các chủ đề thường gặp trong JD và những vấn đề bạn đã gặp khi xây dựng project.*
