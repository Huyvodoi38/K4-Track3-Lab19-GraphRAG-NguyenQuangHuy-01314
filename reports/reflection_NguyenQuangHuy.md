# Báo Cáo Suy Ngẫm Cá Nhân & Kế Hoạch Đồ Án (Reflection & Action Plan)

**Học viên:** Nguyễn Quang Huy  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  

---

## 1. MAPPING BÀI GIẢNG VÀO MÃ NGUỒN

| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|---|---|---|---|
| **Conservative Coreference** | Module 1 | `run_coref()` / Prompt | Giải quyết đại từ nhân xưng/chỉ định theo prompt an toàn, ghi log `unresolved_mentions` để tránh tạo quan hệ giả mạo. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Giữ cho đồ thị luôn sạch, chuẩn hóa theo ontology định sẵn (`Company`, `Person`, `Technology`), loại bỏ quan hệ rác. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Sử dụng câu lệnh Cypher `UNWIND $rows AS row` theo batch, tối ưu I/O và đảm bảo 100% cạnh có đầy đủ `source_chunk_id` và `published_date`. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Gộp hiệu quả các thực thể đồng nghĩa (`ServiceNow Inc.` $\rightarrow$ `ServiceNow`) bằng Vector ANN + Lexical Guard với độ phức tạp tuyến tính. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `recent_edges()` | Cắt tỉa tự động các node có $degree > 100$ về $\le 50$ cạnh mới nhất theo thời gian, ngăn chặn bùng nổ token context. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `judge_json()` | Chấm điểm tự động khách quan trên 3 tiêu chí (Comprehensiveness, Faithfulness, Multi-hop reasoning) kèm giải thích chi tiết. |

---

## 2. QUÁ TRÌNH DEBUG VÀ BÀI HỌC KINH NGHIỆM

### Lỗi kỹ thuật phức tạp nhất đã gặp:
1. **Lệch cấu trúc Schema dữ liệu (KeyError):** File CSV tải về từ Hugging Face có cột nội dung là `description` thay vì `text`. Đã xử lý bằng hàm ánh xạ cột linh hoạt `pick_col()`.
2. **Deprecation Model Groq (Error 404):** Tên model cũ bị ngừng hỗ trợ khiến bước trích xuất ban đầu trả về 0 triples. Đã chuyển sang model 120B mạnh mẽ `openai/gpt-oss-120b` trên Groq và thêm cơ chế Guard bảo vệ DataFrame rỗng.
3. **Neo4j Authentication & Environment:** Cấu hình chuẩn xác `NEO4J_USER=neo4j` và `NEO4J_DATABASE=neo4j` thay vì nhầm với instance ID.

### Bài học cốt lõi rút ra:
- **Kiểm soát AI Coding Agent:** Không phó mặc hoàn toàn cho Agent sinh mã mà phải chủ động audit dữ liệu, kiểm tra độ phức tạp thuật toán và thiết lập các bài test sanity check ở từng giai đoạn.
- **Tầm quan trọng của Provenance:** Việc lưu trữ metadata nguồn gốc (`source_chunk_id`, `published_date`, `evidence`) trên từng cạnh là yếu tố sống còn để biến Knowledge Graph thành công cụ kiểm soát ảo giác (Anti-hallucination) trong môi trường Production.

---

## 3. KẾ HOẠCH ÁP DỤNG VÀO ĐỒ ÁN THỰC TẾ (ACTION PLAN)

- **Tên đồ án / Dự án:** Hệ thống Trợ lý Tri thức Y tế & Dược phẩm Đa nguồn (Medical & Pharma Knowledge Assistant).
- **Lý do cần GraphRAG:**
  - Dữ liệu y khoa và dược phẩm có tính liên kết đa tầng: *Bệnh $\rightarrow$ Triệu chứng $\rightarrow$ Thuốc $\rightarrow$ Hoạt chất $\rightarrow$ Tác dụng phụ $\rightarrow$ Chống chỉ định*.
  - Flat RAG chỉ tìm kiếm theo độ tương đồng câu chữ nên thường xuyên bỏ sót các tương tác thuốc gián tiếp ($A$ tương tác với $B$, $B$ tương tác với $C$) hoặc nhầm lẫn giữa chỉ định và chống chỉ định. GraphRAG cung cấp đường dẫn suy luận logic chính xác và truy vết được phác đồ điều trị.
- **Cấu trúc Ontology (Nodes & Relations) dự kiến:**
  - **Nodes:** `Disease` (Bệnh), `Drug` (Biệt dược), `ActiveIngredient` (Hoạt chất), `SideEffect` (Tác dụng phụ), `ClinicalTrial` (Thử nghiệm lâm sàng).
  - **Relations:** `TREATS`, `CONTAINS`, `CAUSES_SIDE_EFFECT`, `INTERACTS_WITH`, `CONTRAINDICATED_WITH`.
- **Chiến lược xử lý Super-node & Entity Resolution:**
  - *Super-node:* Đối với các bệnh hoặc hoạt chất phổ biến có hàng nghìn liên kết (như *Paracetamol*, *Tăng huyết áp*), phân nhóm cạnh theo mức độ nghiêm trọng (Severity Level) và loại chống chỉ định thay vì chỉ lọc theo thời gian.
  - *Entity Resolution:* Sử dụng từ điển danh mục thuốc của Bộ Y Tế và chuẩn quốc tế (UMLS / MeSH) làm Lexical Guard kết hợp mô hình embedding chuyên ngành BioLinkBERT.

---

## 4. TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|---|:---:|---|
| Mức độ hiểu bài giảng GraphRAG | **5/5** | Nắm vững toàn bộ pipeline từ Preprocessing, Entity Resolution, Graph Traversal đến Evaluation. |
| Khả năng kiểm soát AI Coding Agent | **5/5** | Chủ động audit code, từ chối thuật toán kém tối ưu, debug và cấu hình hệ thống thành công. |
| Chất lượng đồ thị tri thức xây dựng | **5/5** | Đồ thị tuân thủ strict schema, 100% cạnh có provenance đầy đủ. |
| Khả năng phân tích và debug hệ thống | **5/5** | Phân tích thấu đáo nguyên nhân gốc rễ và đưa ra giải pháp mở rộng quy mô rõ ràng. |
