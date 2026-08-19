# Phân Tích Ca Lỗi — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Đặng Nguyên Giáp  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  
**Môi trường:** Jupyter local (Windows, Python 3.12, CPU) + Neo4j AuraDB Free

> File này được tách từ `reports/lab_report.md` để khớp cấu trúc nộp bài
> nêu trong RUBRIC.md. Nội dung hai bản là một.

---

## Hai ca lỗi điển hình

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

## Các lỗi hệ thống khác đã truy vết và xử lý

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

## 📎 PHỤ LỤC: Số liệu tái lập

| Hạng mục | Giá trị |
| --- | --- |
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
