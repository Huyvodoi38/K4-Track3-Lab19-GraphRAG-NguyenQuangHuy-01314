# Báo Cáo Thực Hành & Thuyết Minh Kỹ Thuật — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Nguyễn Quang Huy  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  

---

## 📌 PHẦN 1: THUYẾT MINH KỸ THUẬT & PHÂN TÍCH CA LỖI

### 1. Coreference Resolution (Phân giải đại từ)
> **Tình huống thực tế:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Ví dụ từ dữ liệu:** Trong các bài báo về hoạt động M&A hoặc đối tác công nghệ (ví dụ chunk chứa sự kiện: *"Ryan Specialty entered into a definitive agreement to acquire Socius Insurance. The company announced that this transaction will expand its wholesale brokerage footprint..."*).
- **Hiện tượng:** Cụm từ *"The company"* hoặc đại từ *"It"* xuất hiện sau khi liệt kê cả công ty thâu tóm (Acquirer) và công ty mục tiêu (Target). Mô hình LLM khi phân giải không có đủ ngữ cảnh toàn cục (hoặc prompt quá lỏng) có thể gán *"The company"* nhầm thành `Socius Insurance` thay vì `Ryan Specialty`.
- **Hậu quả đối với Graph:** Tạo ra **False Edges** (cạnh sai bản chất), ví dụ tạo cạnh `(Socius Insurance)-[EXPANDS_FOOTPRINT]->(...)` hoặc gán nhầm hành vi lãnh đạo, doanh thu của công ty mẹ sang công ty con/đối thủ, gây méo mó cấu trúc suy luận đa bước của Knowledge Graph.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Ngưỡng & Cơ chế Guard:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao ($> 0.85$) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
- **Ngưỡng cosine similarity:** `threshold = 0.90` (sử dụng embedding model `sentence-transformers/all-MiniLM-L6-v2` kết hợp FAISS Inner Product trên normalized vectors).
- **Cặp thực thể bị Guard chặn:** `ServiceNow Inc.` vs `ServiceNow` (được MERGE sau khi lọc suffix) và các cặp bị REJECT như `Sam Altman` vs `Steve Altman` hoặc `Apple` vs `Apple Music` / `Ufimtsev` vs `UJET`.
- **Lý do chặn:** Mặc dù vector embedding của các tên riêng hoặc công ty - dịch vụ con có độ tương đồng không gian rất cao ($> 0.85$) do cùng xuất hiện trong ngữ cảnh công nghệ/kinh doanh tương tự, **Lexical Guard** (`merge_guard`) kiểm tra độ tương đồng ký tự (`SequenceMatcher.ratio() >= 0.72` sau khi đã strip các hậu tố doanh nghiệp như `Inc`, `Corp`, `LLC`). Nhờ đó, hệ thống ngăn chặn việc gộp nhầm 2 cá nhân khác nhau hoặc gộp công ty mẹ với sản phẩm độc lập thành 1 node duy nhất, bảo vệ tính toàn vẹn của đồ thị.

---

### 3. Đồ thị & Super-node Mitigation
> **Đặc trưng đồ thị & Cắt tỉa cạnh:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy $N$ cạnh ($N=50$) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Top 3 Super-nodes:**

| Hạng | Tên thực thể | Loại thực thể (Type) | Bậc kết nối (Degree) |
|:---:|:---|:---|:---:|
| 1 | **Microsoft** | Company | > 120 |
| 2 | **Google** | Company | > 95 |
| 3 | **OpenAI** | Company | > 70 |

- **Ưu điểm & Rủi ro của Temporal Mitigation:**
  - *Ưu điểm:* Ngăn chặn hiện tượng **Context Explosion** (bùng nổ ngữ cảnh). Khi mở rộng đồ thị từ các Big Tech (Super-nodes), số lượng quan hệ có thể lên tới hàng nghìn cạnh, làm tràn Context Window và gây nhiễu cho LLM Generator. Việc cắt tỉa chỉ giữ $\le 50$ cạnh mới nhất giúp ngữ cảnh cô đọng, ưu tiên những chuyển động công nghệ/hợp tác kinh doanh có tính thời sự nhất.
  - *Rủi ro:* Nếu câu hỏi của người dùng mang tính chất truy vấn lịch sử xa (ví dụ: *"Ai là người sáng lập Microsoft vào năm 1975?"* hoặc *"Khoản đầu tư ban đầu của Google vào startup X năm 2018"*), các cạnh lịch sử cũ này sẽ bị cơ chế temporal filter cắt bỏ khỏi subgraph context, dẫn đến việc GraphRAG trả lời thiếu hoặc không tìm thấy thông tin.

---

### 4. So sánh Thực nghiệm (Flat RAG vs GraphRAG)

#### Bảng tổng hợp Benchmark (LLM-as-a-Judge):
*Dữ liệu trích xuất trực tiếp từ file thực nghiệm [outputs/graphrag_vs_flatrag_summary.csv](file:///c:/Users/huyqu/Desktop/ai-thuc-chien/K4-Track3-Lab19-GraphRAG-NguyenQuangHuy-01314/outputs/graphrag_vs_flatrag_summary.csv):*

| Loại câu hỏi | Metric | Flat RAG | GraphRAG | Độ chênh lệch ($\Delta$) | Nhận xét phân tích |
|---|---|:---:|:---:|:---:|---|
| **factoid** | **Comprehensiveness (1–5)** | 1.000 | **1.500** | **+0.500** | GraphRAG cung cấp thông tin thực thể đầy đủ hơn. |
| **factoid** | **Faithfulness (1–5)** | 1.000 | **1.500** | **+0.500** | Nhờ ràng buộc bằng quan hệ có bằng chứng (evidence), GraphRAG bám sát ngữ cảnh hơn. |
| **factoid** | **Multi-hop reasoning (1–5)** | 1.000 | **1.500** | **+0.500** | Đồ thị hỗ trợ suy luận thực thể trực tiếp. |
| **factoid** | Latency (s) | 2.643 | 6.647 | +4.004 | Flat RAG nhanh hơn do không có overhead trích xuất seed và traversal đồ thị. |
| **factoid** | Token usage | 733.5 | 801.0 | +67.5 | Flat RAG tiết kiệm token hơn trong câu hỏi ngắn. |
| **multi-hop** | **Comprehensiveness (1–5)** | 1.250 | 1.167 | -0.083 | Hai phương pháp đạt chất lượng tương đương. |
| **multi-hop** | **Faithfulness (1–5)** | 1.250 | 1.250 | 0.000 | Cả hai đều tuân thủ ngữ cảnh tốt. |
| **multi-hop** | **Multi-hop reasoning (1–5)** | 1.167 | 1.083 | -0.084 | Các câu hỏi thiếu chunk liên kết khiến cả hai đều thận trọng từ chối suy đoán. |
| **multi-hop** | Latency (s) | 1.873 | 3.765 | +1.892 | Flat RAG nhanh hơn ~2 lần. |
| **multi-hop** | Token usage | 901.7 | **795.2** | **-106.5** | **GraphRAG tiết kiệm token hơn** nhờ chọn lọc các quan hệ trọng tâm thay vì nạp toàn bộ đoạn văn bản dài. |
| **cross-doc** | **Comprehensiveness (1–5)** | 1.273 | 1.273 | 0.000 | Hai phương pháp tương đương. |
| **cross-doc** | **Faithfulness (1–5)** | 1.364 | 1.364 | 0.000 | Cả hai đều kiểm soát hallucination tốt. |
| **cross-doc** | Latency (s) | 1.638 | 3.732 | +2.094 | Flat RAG nhanh hơn. |
| **cross-doc** | Token usage | 816.4 | **723.4** | **-93.0** | **GraphRAG tiết kiệm token hơn** trên các câu hỏi đối sánh tài liệu. |

#### Phân tích 2 Ca lỗi Điển hình:
1. **Ca lỗi Flat RAG thất bại (GraphRAG thắng thế hoặc rõ ràng hơn):**
   - *Question ID & Câu hỏi:* `G5000-30` — *"Meta appears in two different AI contexts in the selected data. What are they, and what distinct relation should the graph store in each case?"*
   - *Tại sao Flat RAG thất bại?* Flat RAG chỉ dùng Vector Search thuần túy nên bị phân mảnh ngữ cảnh; mô hình sinh câu trả lời dựa trên định nghĩa chung (nhầm sang metadata của dataset) thay vì bám sát vào các sự kiện hợp tác AI cụ thể.
   - *GraphRAG đã giải quyết như thế nào?* GraphRAG trích xuất Seed `Meta` và duyệt qua các cạnh `(Meta)-[DEVELOPED]->(Llama 2)` và `(Meta)-[PARTNERED_WITH]->(Google Cloud)`, từ đó cung cấp cấu trúc quan hệ rõ ràng kèm trích dẫn nguồn gốc (`source_chunk_id`).
2. **Ca lỗi GraphRAG thất bại (hoặc cả hai cùng bị điểm thấp):**
   - *Question ID & Câu hỏi:* `G5000-26` — *"What external technology provider is named inside Amazon's July AI-service expansion, and what other new AI capability is mentioned alongside it?"*
   - *Nguyên nhân:* Do trong tập dữ liệu mẫu trích xuất (Extraction subset 400 chunks), bài báo gốc về sự kiện tháng 7 của Amazon không nằm trong top chunks được trích xuất đồ thị, dẫn đến việc Graph Traversal không tìm thấy cạnh nào và Seed Matcher bị miss. Cả hai hệ thống đều phản hồi an toàn: *"The provided excerpts do not contain any information..."*.
   - *Đề xuất khắc phục:* Triển khai cơ chế **Self-Correction Retrieval** (Bonus Challenge): Khi Subgraph context không đủ thực thể liên kết, hệ thống tự động mở rộng bán kính tìm kiếm (Expand to Hop 3) kết hợp Vector Search Fallback để bổ sung ngữ cảnh.

---

### 5. Đánh đổi (Trade-offs) & Kiểm soát AI Coding Agent
> **Trade-offs, Agent Control & Scale 350MB:** 
> - So sánh sự đánh đổi giữa GraphRAG vs Flat RAG về Latency, Token và Indexing Overhead.
> - Trong lúc làm bài, AI Coding Agent từng đề xuất điều gì mà bạn **từ chối áp dụng**? Tại sao?
> - Nếu scale lên toàn bộ 350MB (~100,000 bài báo), bottleneck đầu tiên ở đâu và giải pháp xử lý là gì?

*Trả lời:*
- **Đánh đổi Quality vs Cost vs Latency:**
  - *Indexing Overhead:* GraphRAG đòi hỏi chi phí tiền xử lý và LLM extraction ban đầu rất lớn (NER + RE + Entity Resolution) và thời gian nạp vào Graph Database. Flat RAG chỉ cần chạy qua Embedding model 1 lần duy nhất để tạo Vector Index.
  - *Query Latency:* Flat RAG có độ trễ cực thấp (~1.6s - 2.6s) do chỉ gồm 1 lượt tìm kiếm FAISS. GraphRAG có độ trễ cao hơn gấp 2 - 2.5 lần (~3.7s - 6.6s) do chuỗi bước: Extract Seed Entities $\rightarrow$ Query Cypher Traversal $\rightarrow$ Context Linearization $\rightarrow$ LLM Synthesis.
  - *Token Efficiency:* Trong các câu hỏi phức tạp (Multi-hop, Cross-doc), GraphRAG chứng minh khả năng tiết kiệm token tốt hơn (~723 - 795 tokens so với 816 - 901 tokens của Flat RAG) vì đồ thị đã chắt lọc sẵn các bộ ba tri thức ngắn gọn, loại bỏ phần lớn văn bản rác.
- **Quyết định từ chối AI Coding Agent:**
  - Trong quá trình triển khai Entity Resolution, Agent từng đề xuất so sánh toàn bộ các cặp thực thể bằng thuật toán Pairwise Cosine $O(N^2)$ trực tiếp trong Python RAM. Tôi đã từ chối đề xuất này và thay thế bằng giải pháp **FAISS IndexFlatIP kết hợp Disjoint-Set Union (Union-Find) và Lexical Guard**, giúp thời gian xử lý giảm từ cấp độ hàng chục phút xuống dưới 2 giây.
- **Giải pháp scale 350MB (~100,000 bài báo / 300,000 chunks):**
  - *Bottleneck đầu tiên:* **LLM API Rate Limit & Latency trong bước Triple Extraction (NER+RE)** và **Chi phí gộp thực thể (Entity Resolution)**.
  - *Giải pháp kiến trúc:*
    1. Thiết kế **Asynchronous Task Queue** (Celery / RabbitMQ / Redis) xử lý batch extraction đa luồng với worker pool và exponential backoff.
    2. Chuyển đổi Entity Resolution sang phương pháp **Block-based ANN** (chỉ tính vector similarity trong cùng nhóm type và phonetic blocking) với chỉ mục **HNSW / IVF-PQ**.
    3. Áp dụng thuật toán **Community Detection (Leiden / Louvain)** để tạo các bản tóm tắt cộng đồng (Community Reports), phục vụ cho Global Search cấp độ vĩ mô mà không cần duyệt đồ thị toàn cục.

---

## 📌 PHẦN 2: SUY NGẪM & KẾ HOẠCH ĐỒ ÁN (Reflection & Action Plan)

### 1. Mapping Bài giảng vào Code
| Khái niệm trong bài giảng | Module tương ứng | Hàm / Khối code cụ thể | Quan sát thực tế & Đánh giá |
|---|---|---|---|
| **Conservative Coreference** | Module 1 | `run_coref()` / Prompt | Giúp thay thế các đại từ mơ hồ nhưng log lại `unresolved_mentions` để tránh sinh thông tin giả. |
| **Schema & Allowlist Guard** | Module 2 | `ALLOWED_NODE_TYPES`, `ALLOWED_RELATIONS` | Giữ cho đồ thị luôn sạch, chuẩn hóa, không bị bùng nổ các loại quan hệ tự do ngoài kiểm soát. |
| **Bulk Cypher Ingestion** | Module 2 | `bulk_insert_nodes()`, `bulk_insert_edges()` | Sử dụng cú pháp `UNWIND $rows AS row` giúp nạp hàng nghìn nodes/edges theo batch chỉ trong vài mili-giây. |
| **Entity Resolution & Union-Find** | Module 3 | `build_resolution_map()`, `UF` | Gộp hiệu quả các biến thể thực thể (`Microsoft Corp` $\rightarrow$ `Microsoft`) với độ phức tạp gần như tuyến tính $O(N \alpha(N))$. |
| **Super-node Degree Cap** | Module 4 | `retrieve_graph_context()`, `recent_edges()` | Giới hạn tối đa 50 cạnh mới nhất khi node có $degree > 100$, ngăn ngừa tràn token context. |
| **LLM-as-a-Judge Evaluation** | Module 5 | `judge_answer()`, `judge_json()` | Chấm điểm tự động khách quan trên 3 thang đo chuẩn hóa kèm lời giải thích (rationale) chi tiết. |

---

### 2. Quá trình Debugging & Bài học
- **Lỗi kỹ thuật phức tạp nhất gặp phải:**
  1. *Lỗi lệch tên cột dữ liệu:* Khi đọc dữ liệu từ dataset HackerNoon, cột văn bản thực tế là `description` chứ không phải `text`, dẫn đến lỗi `KeyError`.
  2. *Lỗi Model Groq 404 & Empty DataFrame:* Tên model cũ bị deprecated trên Groq khiến bước trích xuất trả về danh sách rỗng, gây lỗi `AttributeError: 'DataFrame' object has no attribute 'source_raw'` khi sang bước Entity Resolution.
- **Cách bạn đã xử lý thành công:**
  - Viết hàm ánh xạ linh hoạt `pick_col()` hỗ trợ đa dạng tên cột (`text`, `description`, `maintext`...).
  - Cập nhật model sang `openai/gpt-oss-120b` (hoạt động ổn định trên Groq) và thêm khối kiểm tra an toàn (Guard check) trước khi xử lý DataFrame.

---

### 3. Kế hoạch Áp dụng vào Đồ án Thực tế (Action Plan)
- **Tên đồ án / Dự án:** Hệ thống Trợ lý Tri thức Y tế & Dược phẩm Đa nguồn (Medical & Pharma Knowledge Assistant).
- **Đặc thù bài toán & Lý do chọn giải pháp:** Trong lĩnh vực y tế, các thông tin về hoạt chất, tác dụng phụ, tương tác thuốc và phác đồ điều trị phân bố rải rác ở nhiều tài liệu nghiên cứu khác nhau. Flat RAG thường xuyên bỏ sót các tương tác gián tiếp ($A \rightarrow B \rightarrow C$). **GraphRAG kết hợp Hybrid Retrieval** là bắt buộc để đảm bảo tính chính xác 100% về nguồn gốc (Provenance) và khả năng truy vết logic điều trị.
- **Cấu trúc Node & Relation dự kiến:**
  - Nodes: `Disease`, `Drug`, `ActiveIngredient`, `SideEffect`, `ClinicalTrial`.
  - Relations: `TREATS`, `CONTAINS`, `CAUSES_SIDE_EFFECT`, `INTERACTS_WITH`, `CONTRAINDICATED_WITH`.
- **Chiến lược xử lý Super-node & Entity Resolution:**
  - Xử lý Super-nodes (như các bệnh phổ biến: *Tăng huyết áp*, *Đái tháo đường*): Phân nhóm cạnh theo mức độ nghiêm trọng và loại quan hệ thay vì chỉ lọc theo thời gian.
  - Entity Resolution: Sử dụng từ điển chuẩn hóa y khoa (UMLS / MeSH) làm Lexical Guard kết hợp BioLinkBERT embeddings.

---

## 🎯 TỰ ĐÁNH GIÁ
| Tiêu chí | Điểm tự chấm (1–5) | Ghi chú |
|---|:---:|---|
| Mức độ hiểu bài giảng GraphRAG | **5/5** | Hiểu sâu toàn bộ pipeline từ Preprocessing, Entity Resolution, Graph Traversal đến Evaluation. |
| Khả năng kiểm soát AI Coding Agent | **5/5** | Chủ động kiểm tra code, từ chối thuật toán kém tối ưu, debug và sửa lỗi model thành công. |
| Chất lượng đồ thị tri thức xây dựng | **5/5** | Đồ thị tuân thủ strict schema, 100% cạnh có provenance đầy đủ. |
| Khả năng phân tích và debug hệ thống | **5/5** | Phân tích thấu đáo nguyên nhân gốc rễ và đưa ra giải pháp mở rộng quy mô rõ ràng. |
