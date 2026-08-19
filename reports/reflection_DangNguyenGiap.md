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

> **[PHẦN NÀY CẦN BẠN TỰ ĐIỀN — không ai viết hộ được vì nó là đồ án của bạn]**

- **Tên đồ án / Dự án:** [ĐIỀN]
- **Đặc thù bài toán & Lý do chọn giải pháp:** [ĐIỀN — câu hỏi cần trả lời: dữ liệu của bạn có nhiều **quan hệ giữa các thực thể** trải trên nhiều tài liệu không? Nếu câu hỏi người dùng chủ yếu là tra cứu một sự thật nằm gọn trong một đoạn → Flat RAG là đủ. Chỉ chọn GraphRAG khi có câu hỏi kiểu "A liên quan tới B qua C" mà bằng chứng nằm rải rác.]
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: [ĐIỀN]
  - Relations: [ĐIỀN]
- **Chiến lược xử lý Super-node & Entity Resolution:** [ĐIỀN]

---

## 🎯 TỰ ĐÁNH GIÁ

| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
| ---------- | ------------------- | --------- |
| Mức độ hiểu bài giảng GraphRAG | 5 | Hiểu rõ luồng end-to-end, đã tự phân tích được từng module và minh chứng bằng ví dụ thực nghiệm cụ thể ở cả 5 câu hỏi (kể cả Coreference Resolution, sau khi bổ sung `display()` ở cell 1.7). |
| Khả năng kiểm soát AI Coding Agent | 4 | Đã chủ động từ chối 2 đề xuất "vá nhanh nhưng hạ chuẩn" (tắt TLS verify, nới lỏng validate) và yêu cầu debug tận gốc; vẫn phụ thuộc AI Agent khá nhiều ở khâu viết lại code (cell 1.6 chuyển Groq→OpenAI, fix cell 3.4). |
| Chất lượng đồ thị tri thức xây dựng | 4 | 211 nodes/133 edges, 0 cạnh thiếu provenance; nhưng quy mô còn nhỏ (giới hạn NER+RE ở 100 mẫu) nên chưa kiểm chứng được các cơ chế chống bùng nổ đồ thị (Super-node Mitigation) ở điều kiện thực tế kích hoạt. |
| Khả năng phân tích và debug hệ thống | 5 | Đã tự debug thành công chuỗi lỗi thực tế đa dạng: TLS/certificate, thiếu `load_dotenv()`, sai tên cột dataset, model bị deprecate (Groq 404), hardcoded model override ẩn ở cell 3.4 — mỗi lỗi đều xác định đúng nguyên nhân gốc thay vì chỉ vá triệu chứng. |

---
