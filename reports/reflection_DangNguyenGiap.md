# Suy Ngẫm Cá Nhân & Kế Hoạch Đồ Án — Lab 19

**Học viên:** Đặng Nguyên Giáp  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  
**Môi trường:** Jupyter local (Windows, Python 3.12, CPU) + Neo4j AuraDB Free

> File này được tách từ `reports/lab_report.md` để khớp cấu trúc nộp bài
> nêu trong RUBRIC.md. Nội dung hai bản là một.

---

### 1. Mapping Bài giảng vào Code

| Khái niệm trong bài giảng | Module | Hàm / Khối code | Quan sát thực tế & Đánh giá |
| --- | --- | --- | --- |
| **Conservative Coreference** | M1 | `resolve_coref_batch()`, `run_coref()` | 78/400 chunk có mention không phân giải được. Prompt conservative làm đúng: thà bỏ sót còn hơn tạo False Edge. Chi phí: 99% hạn mức token/ngày. |
| **Schema & Allowlist Guard** | M2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Chặn được quan hệ rác, nhưng ép model chọn nhãn gần đúng khi quan hệ thật không có trong allowlist → sinh cạnh `Airbus -LEADS-> Boeing`. Allowlist quá hẹp cũng là một nguồn lỗi. |
| **Bulk Cypher Ingestion** | M2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | `UNWIND $rows` batch 1000. Nạp 137 node + 90 edge trong 1,2 giây. Ở quy mô này chưa thấy khác biệt so với insert từng dòng, nhưng độ phức tạp mạng khác nhau về bậc. |
| **Entity Resolution & Union-Find** | M3 | `build_resolution_map()`, `UF`, `merge_guard()` | Guard gốc sai cả hai chiều, đã viết lại theo loại thực thể. Union-Find chỉ thực sự cần khi có cụm ≥3 biến thể — dữ liệu này chỉ có cụm 2 phần tử. |
| **Super-node Degree Cap** | M4 | `recent_edges()`, `retrieve_graph_context()` | Không kích hoạt trên dữ liệu thật (degree tối đa 6). Phải viết test tổng hợp mới chứng minh được chính sách hoạt động. |
| **LLM-as-a-Judge** | M5 | `judge_answer()`, `comparison_table()` | Chấm được, nhưng dễ cho điểm cao đồng loạt (ceiling effect). Chỉ phân biệt rõ ở G03 nơi Flat RAG hỏng thật sự. |

---

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)

- **Tên đồ án / Dự án:** Trợ lý hỏi đáp tài liệu nội bộ doanh nghiệp (quy định, quy trình, thông báo nhân sự).

- **Đặc thù bài toán & Lý do chọn giải pháp:** Phần lớn câu hỏi người dùng là tra cứu **một quy định nằm gọn trong một tài liệu** ("Nghỉ phép năm được bao nhiêu ngày?"), rất ít khi phải nối thông tin giữa nhiều văn bản. Với dạng câu hỏi này, thực nghiệm ở lab cho thấy **Flat RAG đạt điểm ngang GraphRAG** (5/5 ở cả 2 câu factoid G01, G02) trong khi rẻ hơn 58% token và nhanh hơn 14%. Chi phí dựng graph đo được là **~400.000 token và ~40 phút cho chỉ 400 chunk**, hoàn toàn không tương xứng với lợi ích.

  → **Kết luận: đồ án của tôi dùng Flat RAG, chưa cần GraphRAG.** Tôi sẽ chỉ cân nhắc bổ sung nhánh graph khi log truy vấn thực tế cho thấy tỉ lệ đáng kể câu hỏi dạng bắc cầu, ví dụ *"Quy định nào thay thế quy định X, và ai ký ban hành nó?"* — đúng dạng câu G03 mà Flat RAG chỉ đạt 1/5 còn GraphRAG đạt 5/5 vì bằng chứng nằm ở hai tài liệu khác nhau.

- **Cấu trúc Node & Relation dự kiến (nếu mở rộng sang Hybrid):**
  - Nodes: `QuyDinh`, `PhongBan`, `NhanSu`, `QuyTrinh`
  - Relations: `BAN_HANH` (PhongBan → QuyDinh), `KY_DUYET` (NhanSu → QuyDinh), `THAY_THE` (QuyDinh → QuyDinh), `AP_DUNG_CHO` (QuyDinh → PhongBan), `THAM_CHIEU` (QuyDinh → QuyDinh)

- **Chiến lược xử lý Super-node & Entity Resolution:**
  - *Super-node:* Node `Phòng Nhân sự` chắc chắn trở thành super-node vì ban hành hầu hết văn bản. Nhưng tôi **không dùng chính sách "50 cạnh mới nhất"** như lab — văn bản nội bộ có hiệu lực dài hạn, quy định năm 2019 vẫn còn áp dụng. Đúng như rủi ro đã phân tích ở Phần 1 mục 3, cắt theo ngày mới nhất sẽ làm mất quy định cũ một cách âm thầm. Thay vào đó: lọc cạnh theo **trạng thái hiệu lực** (bỏ văn bản đã bị `THAY_THE`) rồi mới cắt theo ngày.
  - *Entity Resolution:* Tập thực thể nhỏ, cố định và biết trước (danh sách phòng ban, nhân sự lấy từ HR system). Tôi dùng **bảng alias thủ công** thay vì vector similarity — chính xác tuyệt đối, không có rủi ro false merge kiểu `Sam Altman`/`Steve Altman` đã gặp ở lab. Vector similarity chỉ dùng làm lớp gợi ý để con người duyệt khi xuất hiện tên lạ.

---

## 🎯 TỰ ĐÁNH GIÁ

| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
| --- | --- | --- |
| Mức độ hiểu bài giảng GraphRAG | 5 | Giải thích được vì sao Flat RAG hỏng ở multi-hop bằng ca thực nghiệm cụ thể (G03: bằng chứng nằm ở 2 chunk cách nhau 3 tháng, Flat RAG 1/5 vs GraphRAG 5/5), và định lượng được cái giá phải trả (+58% token truy vấn, ~400k token dựng index). |
| Khả năng kiểm soát AI Coding Agent | 4 | Từ chối 3 đề xuất và có bằng chứng đo lường cho từng quyết định: cách tải 300MB đầu file (gây thiên lệch alphabet, mất Microsoft/Google/Meta), giữ ngưỡng gộp 0.90 (không cặp nào đạt, cặp trùng thật chỉ 0.842), và `assert` super-node trên dữ liệu thật (degree tối đa chỉ 6). Vẫn phụ thuộc AI Agent nhiều ở khâu viết code. |
| Chất lượng đồ thị tri thức xây dựng | 3 | 137 node / 90 edge, 0 cạnh thiếu provenance, audit 55 dòng. Nhưng năng suất trích xuất chỉ ~0,22 triple/chunk và đồ thị còn chứa False Edge (`Airbus -LEADS-> Boeing`) — quy mô quá nhỏ để coi là đồ thị tri thức chất lượng. |
| Khả năng phân tích và debug hệ thống | 5 | Truy vết đúng nguyên nhân gốc cho chuỗi lỗi đa dạng: dataset gated (token hợp lệ vẫn 403), sai tên cột `description`, model `llama-3.3-70b-versatile` đã bị Groq gỡ, JSON mode trả đúng cú pháp nhưng sai schema, và khó nhất là khối `<think>` lẫn vào câu trả lời khiến Judge chấm nhầm vệt suy luận mà không ném ra lỗi nào. |
