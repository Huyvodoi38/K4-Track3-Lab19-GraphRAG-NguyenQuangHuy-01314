# Báo Cáo Thuyết Minh Kỹ Thuật (Technical Defense) — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Nguyễn Quang Huy  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  

---

## 10 CÂU HỎI THUYẾT MINH BẢO VỆ KIẾN TRÚC

### 1. Coreference Resolution (Phân giải đại từ)
> **Câu hỏi:** Nêu ít nhất 1 tình huống cụ thể trong dữ liệu HackerNoon mà cơ chế Coreference Resolution phân giải sai hoặc gặp khó khăn. Hậu quả của nó đối với Knowledge Graph là gì?

*Trả lời:*
- **Tình huống thực tế:** Trong các bài báo tin tức công nghệ về hoạt động M&A hoặc liên minh đối tác (ví dụ: *"Ryan Specialty entered into a definitive agreement to acquire Socius Insurance. The company announced that this transaction will expand its wholesale brokerage footprint..."*).
- **Hiện tượng:** Khi một câu chứa nhiều thực thể (công ty mua và công ty bán), các đại từ hoặc danh từ chung như *"The company"*, *"It"*, *"They"* có thể bị LLM trỏ sai về thực thể gần nhất (`Socius Insurance`) thay vì chủ ngữ trung tâm (`Ryan Specialty`).
- **Hậu quả đối với Knowledge Graph:** Tạo ra **False Edges** (cạnh sai bản chất), gán nhầm sự kiện M&A hoặc năng lực kinh doanh của công ty thâu tóm sang công ty mục tiêu/đối thủ. Khi chạy Multi-hop Graph Traversal, các cạnh sai này dẫn đến suy luận sai lệch hoàn toàn.

---

### 2. Entity Resolution Threshold & Lexical Guard
> **Câu hỏi:** Bạn chọn ngưỡng cosine similarity là bao nhiêu cho vector matching? Trích dẫn 1 cặp thực thể có độ tương đồng vector cao ($> 0.85$) nhưng bị Lexical Guard chặn không cho gộp (Reject) và giải thích lý do.

*Trả lời:*
- **Ngưỡng Cosine Similarity:** `threshold = 0.90` (sử dụng mô hình embedding `sentence-transformers/all-MiniLM-L6-v2`).
- **Cặp thực thể bị Lexical Guard chặn:** `Sam Altman` vs `Steve Altman` hoặc `Apple` vs `Apple Music` (trong dữ liệu thực tế: `Ufimtsev` vs `UJET` với similarity $\approx 0.86$).
- **Lý do chặn:** Mặc dù vector embedding có độ tương đồng không gian rất cao do cùng xuất hiện trong ngữ cảnh công nghệ/kinh doanh, **Lexical Guard** (`merge_guard`) kiểm tra độ tương đồng ký tự `SequenceMatcher.ratio() >= 0.72` sau khi đã strip các hậu tố doanh nghiệp (`Inc`, `Corp`, `LLC`). Nhờ đó, hệ thống ngăn chặn việc gộp nhầm 2 cá nhân khác nhau hoặc gộp công ty mẹ với sản phẩm độc lập thành 1 node duy nhất, bảo vệ tính toàn vẹn của đồ thị.

---

### 3. Super-node Mitigation & Cắt tỉa cạnh
> **Câu hỏi:** Top 3 thực thể có bậc (degree) cao nhất trong đồ thị là gì? Việc ưu tiên lấy $N$ cạnh ($N=50$) có `published_date` mới nhất tại các Super-node mang lại ưu điểm gì và có rủi ro tiềm ẩn nào?

*Trả lời:*
- **Top 3 Super-nodes:**
  1. `Microsoft` (Degree > 120)
  2. `Google` (Degree > 95)
  3. `OpenAI` (Degree > 70)
- **Ưu điểm của Temporal Mitigation:** Ngăn chặn hiện tượng **Context Explosion** (bùng nổ ngữ cảnh). Khi mở rộng đồ thị từ các Big Tech, số lượng quan hệ có thể lên tới hàng nghìn cạnh, làm tràn Context Window và gây nhiễu cho LLM Generator. Việc cắt tỉa chỉ giữ $\le 50$ cạnh mới nhất giúp context cô đọng, ưu tiên những chuyển động công nghệ/hợp tác kinh doanh có tính thời sự nhất.
- **Rủi ro tiềm ẩn:** Nếu câu hỏi của người dùng mang tính chất truy vấn lịch sử xa (ví dụ: *"Ai là người sáng lập Microsoft vào năm 1975?"*), các cạnh lịch sử cũ này sẽ bị cơ chế temporal filter cắt bỏ khỏi subgraph context, dẫn đến việc GraphRAG trả lời thiếu hoặc không tìm thấy thông tin.

---

### 4. Edge Provenance Integrity
> **Câu hỏi:** Tại sao mọi cạnh trong Knowledge Graph bắt buộc phải lưu đầy đủ `source_chunk_id` và `published_date`?

*Trả lời:*
- **Tính kiểm chứng (Verifiability) & Giảm Hallucination:** Giúp LLM Generator trích dẫn nguồn gốc chính xác inline dạng `[chunk_id=...]` trong câu trả lời.
- **Truy vết thời gian (Temporal Reasoning):** Cho phép đồ thị biết được sự kiện diễn ra khi nào để phục vụ cơ chế Super-node pruning và phân tích sự thay đổi quan hệ theo thời gian (Cross-doc timeline).
- **Data Audit & Maintenance:** Khi một bài báo bị chỉnh sửa hoặc gỡ bỏ, hệ thống có thể dễ dàng định vị và xóa/cập nhật chính xác các cạnh xuất phát từ `source_chunk_id` đó trong Neo4j.

---

### 5. Flat RAG Baseline vs GraphRAG Performance
> **Câu hỏi:** Flat RAG và GraphRAG thể hiện như thế nào trên từng nhóm câu hỏi (`factoid`, `multi-hop`, `cross-doc`)?

*Trả lời (theo số liệu thực nghiệm benchmark):*
- **Factoid:** GraphRAG đạt điểm Comprehensiveness/Faithfulness/Multi-hop (1.50) vượt trội so với Flat RAG (1.00) nhờ định vị trực tiếp node và quan hệ rõ ràng.
- **Multi-hop:** Flat RAG đạt điểm trung bình tương đương GraphRAG (1.25 vs 1.17) nhưng **GraphRAG tiết kiệm token hơn** (795 tokens vs 902 tokens của Flat RAG) do đồ thị chắt lọc các bộ ba tri thức ngắn gọn, loại bỏ văn bản thừa.
- **Cross-doc:** Cả hai phương pháp đạt điểm số tương đương (1.27 - 1.36) và GraphRAG tối ưu token hơn (723 vs 816 tokens).

---

### 6. Đánh đổi Chi phí, Thời gian & Tài nguyên (Trade-offs)
> **Câu hỏi:** So sánh sự đánh đổi giữa GraphRAG vs Flat RAG về Latency, Token và Indexing Overhead.

*Trả lời:*
- **Indexing Overhead:** GraphRAG đòi hỏi chi phí tiền xử lý và LLM extraction ban đầu lớn (NER + RE + Entity Resolution) và thời gian nạp vào Graph Database. Flat RAG chỉ cần chạy qua Embedding model 1 lần duy nhất để tạo Vector Index.
- **Query Latency:** Flat RAG có độ trễ cực thấp (~1.6s - 2.6s) do chỉ gồm 1 lượt tìm kiếm FAISS. GraphRAG có độ trễ cao hơn gấp 2 - 2.5 lần (~3.7s - 6.6s) do chuỗi bước: Extract Seed Entities $\rightarrow$ Query Cypher Traversal $\rightarrow$ Context Linearization $\rightarrow$ LLM Synthesis.
- **Token Efficiency:** GraphRAG tiết kiệm token tốt hơn trên các câu hỏi phức tạp vì đồ thị nạp các triple cô đọng thay vì nạp toàn bộ các đoạn văn bản dài.

---

### 7. Kiểm soát AI Coding Agent & Quyết định Kỹ thuật
> **Câu hỏi:** Trong quá trình thực hiện, AI Coding Agent từng đề xuất điều gì mà bạn **từ chối áp dụng**? Tại sao?

*Trả lời:*
- Trong bước Entity Resolution, Agent từng đề xuất tính toán ma trận tương đồng cosine pairwise $O(N^2)$ trên toàn bộ các thực thể trong bộ nhớ RAM Python.
- **Quyết định từ chối:** Thuật toán $O(N^2)$ pairwise thuần túy sẽ gây tràn bộ nhớ (Out-Of-Memory/OOM) và làm chậm nghiêm trọng khi số lượng thực thể tăng lên hàng chục nghìn. Tôi đã chỉ đạo áp dụng **FAISS IndexFlatIP (ANN search)** kết hợp **Disjoint-Set Union (Union-Find)** và **Lexical Guard**, giúp tốc độ gộp thực thể đạt độ phức tạp tuyến tính $O(N \alpha(N))$ và chạy xong trong chưa đầy 2 giây.

---

### 8. Kiến trúc Mở Rộng Dữ Liệu Lớn (Scale Guard & Scale 350MB)
> **Câu hỏi:** Nếu mở rộng toàn bộ dataset 350MB (~100,000 bài báo / 300,000 chunks), bottleneck đầu tiên ở đâu và giải pháp xử lý là gì?

*Trả lời:*
- **Bottleneck chính:** Tốc độ gọi API LLM (Rate Limit & Cost) trong bước trích xuất quan hệ (NER+RE) và bước Entity Resolution trên hàng trăm nghìn thực thể.
- **Giải pháp kiến trúc:**
  1. *Asynchronous Worker Queue:* Xây dựng hàng đợi bất đồng bộ (Celery/RabbitMQ/Redis) xử lý batch extraction đa luồng với worker pool và exponential backoff retry.
  2. *Hierarchical Entity Resolution:* Phân nhóm thực thể theo category/type và phonetic blocking (Double Metaphone) trước khi chạy vector search với chỉ mục **HNSW / IVF-PQ**.
  3. *Community Partitioning:* Áp dụng thuật toán phát hiện cộng đồng (Leiden / Louvain) để tạo Community Summaries phục vụ Global Search cấp độ vĩ mô.

---

### 9. Cypher Bulk Ingestion Performance
> **Câu hỏi:** Vì sao câu lệnh Cypher `UNWIND $rows AS row` theo batch lại là chuẩn bắt buộc trong môi trường Production thay vì chạy từng câu lệnh `CREATE`/`MERGE` đơn lẻ?

*Trả lời:*
- Chạy từng câu lệnh đơn lẻ (`session.run("CREATE ...")`) tạo ra overhead mạng khổng lồ (network roundtrip) và hàng nghìn transaction riêng biệt, làm nghẽn I/O và treo database.
- Cú pháp `UNWIND $rows AS row` cho phép gửi 1 batch hàng trăm nodes/edges trong 1 transaction duy nhất, tận dụng cơ chế thực thi song song của Neo4j engine và chỉ mục duy nhất (`CONSTRAINT ... IS UNIQUE`), tăng tốc độ nạp dữ liệu lên gấp 50–100 lần.

---

### 10. Self-Correction & Community Detection Fallback
> **Câu hỏi:** Khi Graph Traversal không tìm thấy đủ liên kết hoặc bị ngắt quãng, cơ chế Self-Correction Retrieval hoạt động như thế nào?

*Trả lời:*
- Sau khi lấy Subgraph ở Hop 2, một LLM Classifier nhẹ đánh giá xem ngữ cảnh đã đủ để trả lời câu hỏi chưa (`context_sufficient`).
- Nếu chưa đủ: Hệ thống tự động mở rộng bán kính tìm kiếm sang Hop 3.
- Nếu vẫn thiếu thông tin: Hệ thống kích hoạt cơ chế **Vector Fallback** (tăng $k$ chunks từ Flat RAG) hoặc truy vấn vào các **Community Summary Reports** cấp cao để đảm bảo người dùng luôn nhận được câu trả lời đầy đủ và trung thực nhất.
