# Báo Cáo Phân Tích Ca Lỗi (Failure Analysis) — Lab 19: GraphRAG vs Flat RAG

**Học viên:** Nguyễn Quang Huy  
**Khóa học:** AICB-K34 · Track 3: GraphRAG  
**Ngày thực hiện:** 19/08/2026  

---

## 🔍 PHÂN TÍCH CÁC CA LỖI ĐIỂN HÌNH

Dựa trên kết quả đánh giá thực tế từ tập dữ liệu [outputs/graphrag_eval_results.csv](file:///c:/Users/huyqu/Desktop/ai-thuc-chien/K4-Track3-Lab19-GraphRAG-NguyenQuangHuy-01314/outputs/graphrag_eval_results.csv) qua mô hình LLM-as-a-Judge:

---

### Ca Lỗi 1: Flat RAG Thất Bại — GraphRAG Thành Công / Vượt Trội

- **Question ID:** `G5000-30`
- **Loại câu hỏi:** `multi-hop`
- **Nội dung câu hỏi:** *"Meta appears in two different AI contexts in the selected data. What are they, and what distinct relation should the graph store in each case?"*
- **Reference Answer:** *"At Google Cloud Next, Meta is the provider/source of Llama 2 and Code Llama models made available on Google Cloud. Separately, Meta is one of the companies named in the July White House voluntary AI commitments. These are a model-provider/platform relation and a policy/safety-commitment relation, respectively."*

#### 1. Hành vi và Kết quả của Flat RAG:
- *Câu trả lời của Flat RAG:* Mô hình dựa vào Vector Search thuần túy nên bị phân mảnh ngữ cảnh; khi trích xuất chunk chứa từ khóa "Meta", mô hình bị nhầm lẫn giữa định nghĩa công ty Meta và khái niệm "metadata" trong khoa học dữ liệu, sinh ra phân tích lý thuyết chung chung không gắn với sự kiện thực tế trong dataset.
- *Điểm số của Judge:* Comprehensiveness: 2/5, Faithfulness: 2/5, Multi-hop: 2/5.

#### 2. Hành vi và Kết quả của GraphRAG:
- *Câu trả lời của GraphRAG:* Nhờ trích xuất Seed Entity `Meta` và thực hiện Graph Traversal 2 hops, GraphRAG tìm thấy chính xác 2 nhánh quan hệ riêng biệt:
  1. `(Meta)-[DEVELOPED]->(Llama 2)` và `(Meta)-[PARTNERED_WITH]->(Google Cloud)`.
  2. `(Meta)-[PARTNERED_WITH / LEADS]->(White House Commitments)`.
- *Điểm số của Judge:* Cung cấp cấu trúc quan hệ rõ ràng, có trích dẫn nguồn gốc (`source_chunk_id`) minh bạch.

#### 3. Phân tích nguyên nhân gốc rễ (Root Cause):
- Flat RAG dựa vào độ tương đồng vector ngữ nghĩa giữa toàn bộ câu hỏi và các chunk đơn lẻ. Do câu hỏi yêu cầu đối sánh hai bối cảnh xuất hiện khác nhau ở hai văn bản riêng biệt, Vector Search không thể nối ghép được 2 phần thông tin này nếu chúng nằm rải rác.
- GraphRAG khắc phục triệt để nhờ cấu trúc đồ thị: Node `Meta` đóng vai trò là điểm neo trung tâm kết nối các sự kiện khác nhau thông qua các cạnh quan hệ rõ ràng.

---

### Ca Lỗi 2: Ca Lỗi Do Thiếu Extraction Context / Seed Matcher

- **Question ID:** `G5000-26`
- **Loại câu hỏi:** `multi-hop`
- **Nội dung câu hỏi:** *"What external technology provider is named inside Amazon's July AI-service expansion, and what other new AI capability is mentioned alongside it?"*
- **Reference Answer:** *"Amazon's AI-service story names access to technology from Cohere. It also mentions a program for building more conversational customer-service agents; one follow-up additionally mentions a healthcare system for generating clinical notes after patient visits."*

#### 1. Hành vi và Kết quả của cả 2 hệ thống:
- *Câu trả lời:* Cả Flat RAG và GraphRAG đều phản hồi an toàn: *"The provided excerpts do not contain any information about Amazon's July AI-service expansion... I cannot answer based on the given context."*
- *Điểm số của Judge:* Comprehensiveness: 1/5, Faithfulness: 1/5, Multi-hop: 1/5 (Judge đánh giá thấp do không đưa ra được đáp án như Reference).

#### 2. Phân tích nguyên nhân gốc rễ (Root Cause):
- Trong khuôn khổ thời lượng bài lab (Scale Guard), dữ liệu trích xuất đồ thị bị giới hạn ở `EXTRACTION_MAX_CHUNKS = 400` chunks đầu tiên. Bài báo chứa thông tin về sự kiện tháng 7 của Amazon (`Cohere`) không nằm trong top 400 chunks được trích xuất đồ thị này.
- Khi truy vấn, Seed Extractor không tìm thấy cạnh nào liên quan đến Cohere trong Neo4j Subgraph, và Flat RAG top 6 chunks cũng không chứa đoạn văn bản này. Cả hai hệ thống đều chọn giải pháp trung thực là từ chối trả lời (Faithfulness behavior) thay vì bịa đặt (Hallucination).

#### 3. Giải pháp Khắc phục & Cải tiến:
1. **Mở rộng Extraction Chunks:** Nâng `EXTRACTION_MAX_CHUNKS` lên toàn bộ dataset để đồ thị bao phủ 100% dữ liệu.
2. **Self-Correction Graph Retrieval (Bonus):** Khi Subgraph trả về 0 cạnh hoặc không đủ căn cứ, hệ thống tự động fallback mở rộng tìm kiếm Vector Search với $k$ lớn hơn ($k=12$) kết hợp reranking.
