# Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đặng Nguyên Giáp  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  
**Môi trường:** Jupyter local (Windows, Python 3.12, CPU) + Neo4j AuraDB Free

> File này được tách từ `reports/lab_report.md` để khớp cấu trúc nộp bài
> nêu trong RUBRIC.md. Nội dung hai bản là một.

---

### 1. Coreference Resolution (Phân giải đại từ)

- **Ví dụ từ dữ liệu:** chunk `44761f5a508c68e6b4df::c0000` — *"Will holiday hiring and wage hikes offset worries over tech company layoffs?"*. Đại từ **`It`** được ghi vào `unresolved_mentions`, không phân giải được.
- **Hiện tượng:** **78/400 chunk (19,5%)** có ít nhất một mention không phân giải nổi. Hai ca khác: `We` trong chunk `146c430b334199c62ec6::c0000` (bài phỏng vấn, tiền ngữ nằm ở đoạn đã bị cắt), và `the second` trong `2fa36828f526e1cafb05::c0000`.
- **Nguyên nhân gốc:** Dataset HackerNoon dùng cột `description` dài trung bình chỉ **22 từ**. Tiền ngữ của đại từ thường nằm ở thân bài **không có trong dataset**. Prompt conservative đã làm đúng: gặp mơ hồ thì giữ nguyên văn bản gốc thay vì đoán.
- **Hậu quả đối với Graph:** Nếu prompt cố đoán, `It` sẽ bị gán cho công ty được nhắc gần nhất, sinh **False Edge** gắn sự kiện cho sai chủ thể. Đánh đổi ngược lại: giữ nguyên đại từ khiến NER+RE bỏ sót quan hệ — một phần lý do năng suất trích xuất chỉ đạt **~0,22 triple/chunk**.

---

### 2. Entity Resolution Threshold & Lexical Guard

- **Ngưỡng:** `threshold = 0.80` để **gộp**, `audit_threshold = 0.40` để **ghi log**. Hai ngưỡng cố ý tách rời.

- **Vì sao lệch khỏi 0.90 của đề bài:** Đo trên 80 thực thể `Company` cho thấy **không cặp nào đạt 0.90**. Trong khi đó cặp trùng thật `Cisco` ↔ `Cisco Systems` chỉ đo được **0.842**. Giữ ngưỡng 0.90 thì entity resolution **không bao giờ gộp được gì**, đồ thị phân mảnh vĩnh viễn. Hạ xuống 0.80 bắt được cặp này mà vẫn nằm trên cặp nguy hiểm `Airbus` ↔ `Boeing` (0.724).

- **Cặp similarity cao bị Guard chặn:** `Airbus` ↔ `Boeing`, similarity **0.724**. Hai hãng máy bay đối thủ, ngữ cảnh sử dụng gần như giống hệt nên embedding rất gần, nhưng là **hai pháp nhân khác nhau**. Gộp sẽ hợp nhất mọi quan hệ của Airbus vào Boeing.

- **Phát hiện: Guard gốc của đề bài sai cả hai chiều.** Guard gốc dùng `SequenceMatcher(strip_suffix(a), strip_suffix(b)) >= 0.72`:

| Cặp | Ratio | Guard gốc | Đúng ra | Loại lỗi |
| --- | --- | --- | --- | --- |
| `Cisco` ↔ `Cisco Systems` | 0,56 | CHẶN | Gộp | False negative → phân mảnh |
| `Sam Altman` ↔ `Steve Altman` | 0,78 | **GỘP** | Chặn | **False merge → sai thực thể** |

  Trớ trêu là `Sam Altman` vs `Steve Altman` chính là ví dụ ASSIGNMENT.md nêu để cảnh báo, nhưng guard cung cấp sẵn không chặn nổi.

- **Guard đã viết lại — tách luật theo loại thực thể:**
  - *Tổ chức:* loại bỏ từ chung chung (`Systems`, `Group`, `Holdings`, `Technologies`…) rồi so phần lõi. `Cisco` lõi `{cisco}` = `Cisco Systems` lõi `{cisco}` → gộp. `Apple` lõi `{apple}` ≠ `Apple Watch` lõi `{apple, watch}` → chặn.
  - *Người:* bắt buộc trùng **cả họ lẫn tên đầu**. `Sam Altman` và `Steve Altman` cùng họ khác tên đầu → chặn.
  - Kiểm chứng bằng `test_merge_guard()` (cell 5.1): **PASS 7/7 ca**.

- **Bảng audit:** **55 dòng** (54 `REJECT_LOW_SIM`, 1 `MERGE_VECTOR`). Tách `audit_threshold` khỏi ngưỡng gộp là có chủ đích: bảng audit chỉ ghi các cặp *được gộp* thì không còn là audit — phải lưu cả những cặp **đã cân nhắc rồi từ chối** mới truy vết được quyết định.

---

### 3. Đồ thị & Super-node Mitigation

- **Quy mô:** 137 node, 90 edge. Phân bố quan hệ: `USES` 20, `DEVELOPED` 17, `WORKED_AT` 15, `PARTNERED_WITH` 11, `LEADS` 11, `ACQUIRED` 9, `FOUNDED` 3, `INVESTED_IN` 1.

- **Top 3 Super-nodes:**

| Hạng | Tên thực thể | Loại | Bậc |
| ------ | -------------- | ------ | ----- |
| 1 | Cisco | Company | 6 |
| 2 | AI | Technology | 5 |
| 3 | ML / Microsoft | Technology / Company | 4 |

- **Vấn đề:** Degree cao nhất chỉ **6**, cách rất xa ngưỡng `SUPER_NODE_DEGREE = 100`. Nhánh `assert` trong `test_supernode_policy()` gốc **không bao giờ được thực thi** — chính sách không được chứng minh dù code có tồn tại.

- **Cách xử lý:** Viết `test_supernode_policy_synthetic()` — tạo node giả 120 cạnh với `published_date` trải từ 2000-01 đến 2000-04, gọi `recent_edges()`, xác minh rồi xóa sạch:
  - Degree tạo ra 120 → vượt ngưỡng 100 ✓
  - Số cạnh lấy về **50**, đúng `SUPER_NODE_EDGE_CAP` ✓
  - Thứ tự ngày giảm dần, mới nhất `2000-04-30` → cũ nhất `2000-03-12` ✓ (70 cạnh cũ bị cắt đúng thiết kế)

- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - *Ưu điểm:* Chặn bùng nổ context. Không có cap, một node như Google với hàng nghìn cạnh sẽ nuốt trọn cửa sổ ngữ cảnh, đẩy chi phí token lên và làm loãng tín hiệu. Ưu tiên ngày mới phù hợp miền tin tức công nghệ.
  - *Rủi ro:* Câu hỏi lịch sử **mất dữ liệu âm thầm**. Ví dụ hỏi *"Microsoft mua lại công ty nào năm 2016?"* trên node Microsoft có 3000 cạnh — 50 cạnh mới nhất đều thuộc 2024–2025, sự kiện 2016 bị cắt và hệ thống trả lời "không có thông tin" thay vì báo đã cắt bớt. Khắc phục: cấp phát cạnh theo **tầng thời gian** (mỗi năm giữ N cạnh) hoặc lọc theo khoảng thời gian suy ra từ câu hỏi.

---


---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

| Tiêu chí | Flat RAG | GraphRAG | Δ | Nhận xét |
| --- | --- | --- | --- | --- |
| **Comprehensiveness (1–5)** | 4,00 | 5,00 | **+1,00** | GraphRAG bao phủ đủ các vế của câu hỏi multi-hop |
| **Faithfulness (1–5)** | 5,00 | 5,00 | 0,00 | Cả hai bám context, không bịa |
| **Multi-hop Reasoning (1–5)** | 4,00 | 5,00 | **+1,00** | Chênh lệch tập trung hoàn toàn ở G03 |
| **Latency trung bình (s)** | 11,7 | 13,3 | **+14%** | Chi phí truy vấn Neo4j + BFS |
| **Token usage trung bình** | 1.571 | 2.488 | **+58%** | Subgraph linearize làm context phình to |

> **Ghi chú trung thực về phạm vi số liệu:** bảng trên tính trên **4/6 câu (G01–G04)**. G05, G06 (nhóm cross-doc) chưa hoàn tất do cạn hạn mức **200.000 token/ngày** của Groq free tier trên toàn bộ các model. Xem mục 5 về ràng buộc quota này.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent

**Đánh đổi Quality vs Cost vs Latency**

Chi phí **truy vấn**: GraphRAG +58% token, +14% latency. Nhưng con số đáng chú ý hơn nằm ở chi phí **dựng index**:

| | Flat RAG | GraphRAG |
| --- | --- | --- |
| Dựng index | Embedding 1.500 chunk trên CPU: **11 giây**, 0 token API | Coref + NER/RE 400 chunk: **~40 phút**, **~400.000 token** |
| Mỗi truy vấn | 1.571 token | 2.488 token |

Riêng coref đã ngốn **197.976/200.000 token/ngày** của `gpt-oss-120b` — 99% hạn mức ngày chỉ để chuẩn hóa đại từ, **chưa trích xuất được tri thức nào**. Đây là chi phí ẩn lớn nhất của GraphRAG và nó không xuất hiện trong bất kỳ bảng benchmark truy vấn nào.

Kết luận: ở quy mô 1.500 chunk văn bản ngắn, GraphRAG **chỉ xứng đáng cho nhóm câu hỏi multi-hop**. Với factoid, Flat RAG cho chất lượng ngang bằng (5/5 cả hai) với chi phí thấp hơn nhiều — dùng GraphRAG cho factoid là lãng phí thuần túy.

**Quyết định từ chối đề xuất của AI Coding Agent**

1. **Từ chối cách tải dữ liệu theo prefix.** Code mẫu stream tuần tự và dừng ở 300MB đầu — nhanh và đơn giản. Tôi từ chối vì file gốc nặng **2005MB và được sắp xếp theo `companyName`** (dòng đầu là `01Synergy`): lấy 300MB đầu chỉ cho ra công ty vần **0–B**, tức **không hề có Microsoft, Google, Meta** — đúng những thực thể mà câu hỏi Golden hỏi tới. Benchmark khi đó đo sai hoàn toàn: GraphRAG trượt vì thiếu dữ liệu chứ không phải vì kiến trúc kém. Thay bằng **HTTP Range 60 lát rải đều** khắp file: tải 120MB trong 116 giây, phủ từ `01Synergy` đến `vKirirom`, 430 công ty.
2. **Từ chối giữ ngưỡng gộp 0.90 theo đề bài** sau khi đo thấy không cặp nào trong dữ liệu thật đạt ngưỡng đó, trong khi cặp trùng thật chỉ đạt 0.842 (mục 2).
3. **Từ chối `assert` super-node trên dữ liệu thật** vì degree tối đa chỉ 6 nên assert không bao giờ chạy — thay bằng test tổng hợp có kiểm soát (mục 3).

**Giải pháp scale lên 350MB (~100.000 bài báo)**

- **Bottleneck đầu tiên là hạn mức token của LLM, không phải CPU hay RAM.** Ngoại suy tuyến tính: 400 chunk tốn ~400k token, vậy 100.000 bài ≈ **100 triệu token**. Với hạn mức 200k/ngày cần **500 ngày**. Đây là rào chắn cứng, phải xử lý trước mọi thứ khác.
  - *Hướng xử lý:* (a) Lọc trước bằng heuristic rẻ tiền — chỉ đưa qua LLM những chunk chứa thực thể đã biết hoặc động từ quan hệ (`acquired`, `invested`, `partnered`), ước tính loại được 70–80% chunk vô ích; (b) chạy NER bằng model cục bộ (spaCy/GLiNER) rồi chỉ dùng LLM cho bước RE trên các cặp thực thể đã tìm được; (c) model nhỏ cho chunk đơn giản, model lớn chỉ cho chunk phức tạp.
- **Bottleneck thứ hai: Entity Resolution O(N²).** 100k bài sinh khoảng 500k mention; so sánh toàn cặp là bất khả thi. Giải pháp: **blocking** theo ký tự đầu và loại thực thể, kết hợp index **HNSW** thay `IndexFlatIP`, chỉ so trong từng block.
- **Bottleneck thứ ba: super-node thật sự xuất hiện.** Ở quy mô này Google/Microsoft sẽ có hàng chục nghìn cạnh, lúc đó chính sách cap 50 mới thực sự phát huy tác dụng — và rủi ro mất dữ liệu lịch sử nêu ở mục 3 trở thành vấn đề nghiêm trọng, cần chuyển sang cấp phát theo tầng thời gian.
- **Ingestion:** giữ `UNWIND` batch 1000 nhưng chuyển sang worker queue bất đồng bộ; Neo4j AuraDB Free giới hạn dung lượng nên phải nâng tier hoặc self-host.

---
