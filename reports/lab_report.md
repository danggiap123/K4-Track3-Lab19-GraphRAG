# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** [ĐIỀN HỌ TÊN]
**Khóa học:** AICB-K34 · Track 3: GraphRAG
**Ngày thực hiện:** 19/08/2026

**Môi trường:** Jupyter local (Windows, Python 3.12, CPU) + Neo4j AuraDB Free
**Model:** Coref/Extraction `openai/gpt-oss-120b` · Generator `qwen/qwen3.6-27b` · Judge `qwen/qwen3.6-27b` (qua Groq)

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

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
|---|---|---|---|---|
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
|------|--------------|------|-----|
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

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

| Tiêu chí | Flat RAG | GraphRAG | Δ | Nhận xét |
|---|---|---|---|---|
| **Comprehensiveness (1–5)** | 4,00 | 5,00 | **+1,00** | GraphRAG bao phủ đủ các vế của câu hỏi multi-hop |
| **Faithfulness (1–5)** | 5,00 | 5,00 | 0,00 | Cả hai bám context, không bịa |
| **Multi-hop Reasoning (1–5)** | 4,00 | 5,00 | **+1,00** | Chênh lệch tập trung hoàn toàn ở G03 |
| **Latency trung bình (s)** | 11,7 | 13,3 | **+14%** | Chi phí truy vấn Neo4j + BFS |
| **Token usage trung bình** | 1.571 | 2.488 | **+58%** | Subgraph linearize làm context phình to |

> **Ghi chú trung thực về phạm vi số liệu:** bảng trên tính trên **4/6 câu (G01–G04)**. G05, G06 (nhóm cross-doc) chưa hoàn tất do cạn hạn mức **200.000 token/ngày** của Groq free tier trên toàn bộ các model. Xem mục 5 về ràng buộc quota này.

#### Phân tích 2 Ca lỗi Điển hình

**1. Ca Flat RAG thất bại (GraphRAG thành công) — G03**

- *Câu hỏi:* "Which company has a partnership with Cisco, and what AI chatbot product did that same company develop? Give both relations with their dates."
- *Điểm số:* Flat RAG **1 / 5 / 1** (Comp / Faith / Multi-hop) — GraphRAG **5 / 5 / 5**
- *Tại sao Flat RAG thất bại:* Bằng chứng nằm ở **hai chunk khác nhau, cách nhau hơn 3 tháng**: `17485ee9…` (2023-01-31, hợp tác Cisco–Microsoft) và `d2dcbbd4…` (2023-05-04, Microsoft phát triển Bing AI Chatbot). Không chunk nào chứa đồng thời cả hai vế. Vector search xếp hạng theo độ tương đồng với **toàn bộ câu hỏi**, nên chunk nhiều từ khóa "Cisco" + "partnership" được ưu tiên, còn chunk về Bing AI Chatbot bị đẩy khỏi top-6 vì không nhắc gì tới Cisco.
- *GraphRAG giải quyết thế nào:* Seed extraction lấy ra `Cisco`, BFS 2 hop đi qua `Cisco -PARTNERED_WITH-> Microsoft -DEVELOPED-> Bing AI Chatbot`. Quan hệ là dữ liệu hạng nhất nên phép nối không phụ thuộc việc hai sự kiện có tình cờ nằm chung một đoạn văn hay không. Câu trả lời kèm provenance đầy đủ:
  > *"Microsoft partnered with Cisco on 2023-01-31 [chunk_id=17485ee9c154bc774621::c0000]. Microsoft developed the Bing AI Chatbot on 2023-05-04 [chunk_id=d2dcbbd4fcfab4d9906a::c0000]."*

**2. Ca GraphRAG thất bại — False Edge từ bước Relation Extraction**

- *Hiện tượng:* Đồ thị chứa cạnh `Airbus -LEADS-> Boeing`, sinh từ đoạn gốc *"pulls further ahead of Boeing"* (chunk `d789325048afc2e09dee::c0000`, 2023-06-24).
- *Nguyên nhân:* Model hiểu "leads" theo nghĩa **dẫn trước đối thủ trong cạnh tranh** rồi ánh xạ vào quan hệ `LEADS` trong allowlist vốn mang nghĩa **lãnh đạo/đứng đầu**. Schema allowlist ép model chọn một trong 8 nhãn cho sẵn, nên khi quan hệ thật (cạnh tranh) không có trong danh sách, model chọn nhãn gần nhất về từ vựng thay vì từ chối trích xuất.
- *Ca tương tự:* `Boeing -WORKED_AT-> Air Force One` — câu gốc nói về nhân viên Boeing mất thẻ an ninh khi làm việc trên chuyên cơ tổng thống. Chiều quan hệ bị đảo, và `Air Force One` (một chiếc máy bay) bị gán nhãn `Company`.
- *Hậu quả:* Hai cạnh sai làm nhiễu câu G06 vốn hỏi về quan hệ cạnh tranh Boeing–Airbus.
- *Đề xuất khắc phục:* (a) Bổ sung nhãn `COMPETES_WITH` vào allowlist để model có chỗ đặt quan hệ cạnh tranh; (b) xác thực chiều quan hệ — kiểm tra `source_type`/`target_type` có hợp lệ không (`WORKED_AT` phải là `Person → Company`, không được `Company → Company`); (c) đặt ngưỡng `confidence` tối thiểu, đưa cạnh dưới ngưỡng vào hàng chờ review.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent

**Đánh đổi Quality vs Cost vs Latency**

Chi phí **truy vấn**: GraphRAG +58% token, +14% latency. Nhưng con số đáng chú ý hơn nằm ở chi phí **dựng index**:

| | Flat RAG | GraphRAG |
|---|---|---|
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

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN

### 1. Mapping Bài giảng vào Code

| Khái niệm trong bài giảng | Module | Hàm / Khối code | Quan sát thực tế & Đánh giá |
|---|---|---|---|
| **Conservative Coreference** | M1 | `resolve_coref_batch()`, `run_coref()` | 78/400 chunk có mention không phân giải được. Prompt conservative làm đúng: thà bỏ sót còn hơn tạo False Edge. Chi phí: 99% hạn mức token/ngày. |
| **Schema & Allowlist Guard** | M2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Chặn được quan hệ rác, nhưng ép model chọn nhãn gần đúng khi quan hệ thật không có trong allowlist → sinh cạnh `Airbus -LEADS-> Boeing`. Allowlist quá hẹp cũng là một nguồn lỗi. |
| **Bulk Cypher Ingestion** | M2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND $rows` batch 1000. Nạp 137 node + 90 edge trong 1,2 giây. Ở quy mô này chưa thấy khác biệt so với insert từng dòng, nhưng độ phức tạp mạng khác nhau về bậc. |
| **Entity Resolution & Union-Find** | M3 | `build_resolution_map()`, `UF`, `merge_guard()` | Guard gốc sai cả hai chiều, đã viết lại theo loại thực thể. Union-Find chỉ thực sự cần khi có cụm ≥3 biến thể — dữ liệu này chỉ có cụm 2 phần tử. |
| **Super-node Degree Cap** | M4 | `recent_edges()`, `retrieve_graph_context()` | Không kích hoạt trên dữ liệu thật (degree tối đa 6). Phải viết test tổng hợp mới chứng minh được chính sách hoạt động. |
| **LLM-as-a-Judge** | M5 | `judge_answer()`, `comparison_table()` | Chấm được, nhưng dễ cho điểm cao đồng loạt (ceiling effect). Chỉ phân biệt rõ ở G03 nơi Flat RAG hỏng thật sự. |

---

### 2. Quá trình Debugging & Bài học

**Lỗi kỹ thuật phức tạp nhất:** Câu trả lời chứa nguyên khối `<think>` của reasoning model.

- *Bối cảnh:* Do cạn quota, generator phải chuyển sang `qwen3.6-27b` — một reasoning model xuất khối `<think>...</think>` trước câu trả lời thật.
- *Triệu chứng đánh lừa:* Pipeline chạy **thành công**, xuất đủ CSV, bảng benchmark toàn điểm 5/5 trông rất đẹp. Không có lỗi nào được ném ra.
- *Cách phát hiện:* Chỉ khi mở từng câu trả lời thô ra đọc mới thấy toàn bộ nội dung là vệt suy luận. Câu G05 bị chấm 1/1/1 vì khối `<think>` ăn hết `max_tokens`, câu trả lời thật chưa kịp xuất hiện thì bị cắt.
- *Xử lý:* Lọc `<think>` trong `generate_answer()` (xử lý cả trường hợp khối think bị cắt dở), đồng thời điều chỉnh `max_tokens`.
- **Bài học quan trọng nhất của cả buổi lab:** *Pipeline chạy không lỗi không có nghĩa là kết quả đúng.* Một khác biệt tưởng như vô hại giữa hai model — có hay không sinh reasoning trace — đủ sức làm sai lệch toàn bộ kết quả đánh giá mà không ném ra bất kỳ exception nào. Khi đổi sang họ model khác, phải rà lại **mọi** điểm tiêu thụ output, chứ không chỉ điểm vừa gặp lỗi.

**Các lỗi khác đã xử lý:**

- Dataset gated trên Hugging Face: `HF_TOKEN` hợp lệ vẫn bị 403 — token và quyền truy cập là hai thứ khác nhau, phải bấm "Agree" trên trang dataset.
- `pick_col()` không nhận cột `description` của dataset thật → `KeyError` ngay bước load.
- JSON mode chỉ đảm bảo **cú pháp** hợp lệ, không đảm bảo **schema** đúng: model trả `relations: ["chuỗi"]` thay vì list object làm chết cả cell vì đoạn parse nằm ngoài khối `try`.
- `run_evaluation()` luôn chạy lại từ câu đầu, đốt quota cho các câu đã có kết quả → thêm cơ chế resume từ checkpoint.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)

> **[PHẦN NÀY CẦN BẠN TỰ ĐIỀN — không ai viết hộ được vì nó là đồ án của bạn]**

- **Tên đồ án / Dự án:** [ĐIỀN]
- **Đặc thù bài toán & Lý do chọn giải pháp:** [ĐIỀN — câu hỏi cần trả lời: dữ liệu của bạn có nhiều **quan hệ giữa các thực thể** trải trên nhiều tài liệu không? Nếu câu hỏi người dùng chủ yếu là tra cứu một sự thật nằm gọn trong một đoạn → Flat RAG là đủ. Chỉ chọn GraphRAG khi có câu hỏi kiểu "A liên quan tới B qua C" mà bằng chứng nằm rải rác.]
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: [ĐIỀN]
  - Relations: [ĐIỀN]
- **Chiến lược xử lý Super-node & Entity Resolution:** [ĐIỀN]

---

## 🎯 TỰ ĐÁNH GIÁ

> **[CẦN BẠN TỰ CHẤM]**

| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|----------|-------------------|---------|
| Mức độ hiểu bài giảng GraphRAG | | |
| Khả năng kiểm soát AI Coding Agent | | |
| Chất lượng đồ thị tri thức xây dựng | | |
| Khả năng phân tích và debug hệ thống | | |

---

## 📎 PHỤ LỤC: Số liệu tái lập

| Hạng mục | Giá trị |
|---|---|
| Nguồn dữ liệu | `cleanedCompanyNews.csv`, 2005 MB, tải bằng HTTP Range 60 lát |
| Dòng thu được | 95.643 (bỏ 102.073 dòng vỡ khi cắt byte) · 430 công ty · 57,8 MB |
| Sau exact dedup (SHA-1) | 95.588 → 86.473 |
| Lấy mẫu theo scale guard | 1.500 bài → **1.500 chunk** (1 bài = 1 chunk vì `description` chỉ ~22 từ) |
| Chunk đưa qua extraction | 400 |
| Triples trích xuất | 90 (~0,22 triple/chunk) |
| Đồ thị Neo4j | **137 node · 90 edge** |
| `invalid_provenance_edges` | **0** |
| Bảng audit entity resolution | **55 dòng** (54 `REJECT_LOW_SIM`, 1 `MERGE_VECTOR`) |
| `test_merge_guard()` | **PASS 7/7** |
| `test_supernode_policy_synthetic()` | **PASS** — degree 120 → 50 cạnh, ưu tiên mới nhất |
