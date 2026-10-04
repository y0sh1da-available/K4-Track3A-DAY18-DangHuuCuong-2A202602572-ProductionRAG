# Individual Reflection — Lab 18: Production RAG

**Họ và tên:** Đặng Hữu Cương  
**Khóa:** K4 - Track 3A  
**MSSV:** 2A202602572  
**Ngày hoàn thành:** 04/10/2026  

---

## Phần 1: Mapping bài giảng (Lecture Mapping)

Bảng đối chiếu toàn diện giữa các nguyên lý lý thuyết trong bài giảng và hiện thực mã nguồn trong Lab 18:

| Lecture Concept | Module | Hàm cụ thể | Observation & Phân tích |
|----------------|--------|-------------|--------------------------|
| **Semantic Chunking** | M1 Chunking | `chunk_semantic()` | Sử dụng mô hình sentence embedding `all-MiniLM-L6-v2` tính cosine similarity giữa các câu liên tiếp. Ngưỡng tương đồng `SEMANTIC_THRESHOLD = 0.85` giúp nhóm các câu cùng chủ đề mạch lạc, giải quyết triệt để vấn đề "cắt ngang xương" giữa câu của Naive Paragraph Chunking. |
| **Hierarchical Chunking (Parent-Child)** | M1 Chunking | `chunk_hierarchical()` | Tách tài liệu thành các khối lớn Parent (2048 ký tự) và khối nhỏ Child (256 ký tự). Đảm bảo mỗi child mang `parent_id` liên kết chính xác về parent. Thiết kế này tối ưu hóa độ chính xác tìm kiếm (search trên child gọn gàng) nhưng vẫn cung cấp đầy đủ ngữ cảnh cho LLM đọc (trả về parent). |
| **Structure-Aware Chunking** | M1 Chunking | `chunk_structure_aware()` | Parse tài liệu dựa trên Markdown headers (`#`, `##`, `###`), bảo toàn nguyên vẹn danh sách, bảng biểu và gán metadata `section`. Giúp vector search phân định rõ ranh giới logic của văn bản quy chế. |
| **Vietnamese Word Segmentation** | M2 Search | `segment_vietnamese()` | Sử dụng `underthesea.word_tokenize` kết hợp chuẩn hóa thay thế dấu gạch nối `_` thành khoảng trắng. Khắc phục triệt để lỗi bất đối xứng token giữa query không dấu gạch nối ("nghỉ phép") và từ ghép trong chỉ mục ("nghỉ_phép"). |
| **BM25 + Dense Fusion (RRF)** | M2 Search | `reciprocal_rank_fusion()` | Áp dụng công thức chuẩn $RRF\_Score(d) = \sum \frac{1}{k + rank(d) + 1}$ với $k=60$. Hòa trộn ưu điểm của từ khóa chính xác (BM25 - Lexical) và hiểu nghĩa trừu tượng (Dense - `BAAI/bge-m3`), bù trừ khuyết điểm của từng phương pháp khi truy vấn từ viết tắt, số hiệu hoặc tên riêng. |
| **Cross-Encoder Reranking** | M3 Rerank | `CrossEncoderReranker.rerank()` | Sử dụng mô hình `BAAI/bge-reranker-v2-m3` đánh giá tương tác chéo (cross-attention) toàn diện giữa query và top-20 candidate documents, cô đọng thành top-3 chất lượng cao nhất trước khi đưa vào context window của LLM. |
| **RAGAS 4 Metrics & Diagnostic Tree** | M4 Eval | `evaluate_ragas()`, `failure_analysis()` | Hiện thực hóa khung đo lường RAGAS gồm Faithfulness, Answer Relevancy, Context Precision, Context Recall. Xây dựng cây chẩn đoán (Diagnostic Tree) để tự động ánh xạ điểm số yếu nhất sang nguyên nhân gốc (hallucination, missing chunks, irrelevant noise) và giải pháp khắc phục. |
| **Contextual Prepend & Enrichment** | M5 Enrichment | `contextual_prepend()`, `_enrich_single_call()` | Hiện thực kỹ thuật làm giàu văn bản theo đề xuất của Anthropic (Contextual Embeddings). Đạt tiêu chuẩn tối ưu chi phí qua hàm single-call (1 prompt sinh đồng thời Summary + Questions + Context + Metadata) kèm cơ chế extractive fallback mượt mà khi không có API key. |

---

## Phần 2: Khó khăn & Cách giải quyết (Challenges & Debugging)

Trong quá trình thực hiện bài cá nhân, em đã gặp và xử lý các vấn đề kỹ thuật sau:

1. **Lỗi thiếu thư viện pypdf và sentence-transformers khi khởi tạo:**
   - **Exact Error:** `ModuleNotFoundError: No module named 'pypdf'`, `ModuleNotFoundError: No module named 'sentence_transformers'`.
   - **Nguyên nhân & Debug:** Môi trường Python hệ thống chưa được cài đặt các gói trong `requirements.txt`.
   - **Cách xử lý:** Kích hoạt cài đặt đầy đủ bộ package thông qua `pip install -r requirements.txt`. Đảm bảo các thư viện chuyên sâu như `underthesea`, `qdrant-client`, `flashrank`, `ragas` được nạp đúng phiên bản.

2. **Lỗi mã hóa console Windows khi in văn bản tiếng Việt:**
   - **Exact Error:** `UnicodeEncodeError: 'charmap' codec can't encode characters in position 10-12: character maps to <undefined>`.
   - **Nguyên nhân & Debug:** Shell mặc định trên Windows PowerShell dùng bảng mã cp1252 khiến lệnh in tiếng Việt có dấu qua `sys.stdout` bị crash.
   - **Cách xử lý:** Tận dụng cấu hình `sys.stdout.reconfigure(encoding="utf-8")` và `sys.stderr.reconfigure(encoding="utf-8")` ở đầu tất cả các script Python, giúp mã nguồn chạy ổn định xuyên suốt không lỗi font.

3. **Vấn đề phân tách từ ghép của underthesea trong BM25 Index:**
   - **Hiện tượng:** Khi query "nghỉ phép", BM25 không tìm thấy tài liệu nếu underthesea tokenize tài liệu thành "nghỉ_phép".
   - **Cách xử lý:** Trong hàm `segment_vietnamese()`, sau khi tokenize bằng underthesea, thực hiện `.replace("_", " ")` để đưa toàn bộ token về dạng từ đơn nhất quán cho cả corpus và query.

4. **Tối ưu hóa thời gian tải model & Tránh Deprecation:**
   - **Hiện tượng:** Mỗi lần gọi hàm lại tải lại mô hình embedding hoặc reranker gây suy giảm hiệu năng nghiêm trọng; `QdrantClient.recreate_collection` phát cảnh báo deprecation.
   - **Cách xử lý:** Áp dụng singleton/lazy loading pattern (`_get_semantic_model()`, `_get_encoder()`, `_load_model()`). Với Qdrant, chuyển sang kiểm tra `collection_exists()`, `delete_collection()` và `create_collection()` kết hợp in-memory fallback `:memory:` khi chưa khởi động Docker daemon.

---

## Phần 3: Action Plan cho Project cá nhân (Application Plan)

### Project: Hệ thống Trợ lý Hỏi đáp Quy chế & Pháp lý Doanh nghiệp (Enterprise Policy RAG)

#### 1. Hiện trạng
- **Pipeline hiện tại:** Sử dụng Naive RAG cơ bản với Chunking theo độ dài cố định (500 ký tự), chỉ dùng Dense Search thuần túy trên mô hình embedding đa ngôn ngữ chung chung, không có cơ chế Reranking hay Metadata Filtering.
- **Vấn đề / Bottlenecks đang gặp:**
  - *Context Fragmentation:* Các điều khoản quy định bị cắt đôi giữa chừng làm mất điều kiện ràng buộc.
  - *Temporal Conflict:* Hệ thống trả lời nhầm các văn bản quy chế cũ đã hết hiệu lực thay vì văn bản mới ban hành.
  - *Low Precision on Numbers:* Khả năng tra cứu số liệu hạn mức tài chính, ngày phép, mức chi công tác phí thiếu chính xác.

#### 2. Kế hoạch cải tiến
1. **Chunking Strategy:** Chuyển sang kết hợp **Structure-Aware Chunking** (bảo toàn cấu trúc Chương - Điều - Mục trong văn bản pháp lý) và **Hierarchical Chunking** (Parent 2048 tokens - Child 256 tokens). Khi match vào điều khoản cụ thể, LLM sẽ nhận được toàn bộ nội dung của Điều đó.
2. **Search Retrieval:** Ứng dụng **Hybrid Search kết hợp Reciprocal Rank Fusion (RRF)**:
   - BM25 với `underthesea` để bắt chính xác số hiệu văn bản (ví dụ "Nghị định 13/2023", "Quyết định 45/QĐ").
   - Dense Search với `BAAI/bge-m3` để bắt ngữ nghĩa tổng thể.
   - Thêm bộ lọc Metadata bắt buộc `status: "active"` để loại bỏ văn bản hết hiệu lực.
3. **Reranking:** Tích hợp `BAAI/bge-reranker-v2-m3` để xếp hạng lại top-25 candidate về top-3 chất lượng nhất, giảm thiểu tối đa context thừa đưa vào LLM.
4. **Enrichment:** Sử dụng kỹ thuật **Contextual Prepend** theo phong cách Anthropic (gắn metadata về Tên văn bản, Số hiệu, Ngày ban hành, Chương mục vào đầu mỗi chunk trước khi embed).
5. **Evaluation:** Thiết lập bộ benchmark tự động bằng **RAGAS** với 4 chỉ số vàng (Faithfulness ≥ 0.85, Context Recall ≥ 0.80) chạy định kỳ mỗi khi cập nhật cơ sở dữ liệu tri thức.

#### 3. Timeline triển khai
- **Tuần 1:**
  - Chuẩn hóa lại toàn bộ corpus văn bản pháp lý & quy chế công ty dưới định dạng Markdown chuẩn có metadata header.
  - Triển khai Structure-Aware & Hierarchical Chunking.
- **Tuần 2:**
  - Xây dựng cụm Qdrant Vector DB & Elasticsearch/BM25 tiếng Việt.
  - Hiện thực Hybrid Search với thuật toán RRF.
- **Tuần 3:**
  - Tích hợp lớp Cross-Encoder Reranker (`bge-reranker-v2-m3`).
  - Triển khai Contextual Prepend & Auto-Metadata enrichment.
- **Tuần 4:**
  - Viết bộ 50 câu hỏi kiểm thử đặc thù (Golden Testset).
  - Chạy đánh giá RAGAS, thiết lập CI/CD pipeline kiểm soát chất lượng tự động trước khi deploy Production.
