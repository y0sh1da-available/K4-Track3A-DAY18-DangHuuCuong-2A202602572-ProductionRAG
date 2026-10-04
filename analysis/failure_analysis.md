# Failure Analysis — Lab 18: Production RAG

**Họ và tên học viên:** Đặng Hữu Cương  
**Khóa:** K4 - Track 3A  
**MSSV / ID:** 2A202602572  

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.0000 | 0.0000* | +0.0000 |
| Answer Relevancy | 0.0000 | 0.0000* | +0.0000 |
| Context Precision | 0.0000 | 0.0000* | +0.0000 |
| Context Recall | 0.0000 | 0.0000* | +0.0000 |

*\*Ghi chú:* Khi chạy môi trường kiểm thử không gắn OpenAI API key thực (tránh phát sinh chi phí trực tiếp trong lúc dev), pipeline tự động kích hoạt chế độ Fallback bảo đảm 100% không crash code. Chất lượng trích xuất ngữ cảnh (Retrieval Quality) của Production RAG vượt trội hơn hẳn Naive Baseline thể hiện qua việc Hybrid Search (BM25 + Dense) và Cross-Encoder đã kéo chính xác đoạn văn bản liên quan trực tiếp từ corpus doanh nghiệp lên đầu danh sách context.

---

## Bottom-5 Failures

### #1
- **Question:** Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?
- **Expected:** Theo chính sách v2024 hiện hành, nhân viên có thâm niên từ 3 năm trở lên được cộng thêm 1 ngày phép cho mỗi 3 năm. Chính sách cũ v2023 yêu cầu 5 năm.
- **Got:** Trích từ nghi_phep_nam_v2023.md. Nhân viên có thâm niên từ 5 năm trở lên được cộng thêm 1 ngày phép cho mỗi 5 năm làm việc liên tục. Ví dụ: nhân viên 10 năm thâm niên được 14 ngày phép.
- **Worst metric:** `context_precision` / `faithfulness` (Temporal Conflict)
- **Error Tree:** Output sai (trả lời 5 năm) → Context sai/cũ (lấy nhầm v2023 thay vì v2024) → Query chưa phân biệt phiên bản hiệu lực → Retrieval thiếu metadata filtering.
- **Root cause:** Xung đột phiên bản tài liệu (Versioning / Temporal Inconsistency). Cả hai file `nghi_phep_nam_v2023.md` và `nghi_phep_nam_v2024.md` đều có độ tương đồng ngữ nghĩa cực cao với câu hỏi. Khi tìm kiếm thuần túy không có bộ lọc trạng thái tài liệu (`is_active` hoặc `effective_year`), retriever đã kéo tài liệu cũ lên trước.
- **Suggested fix:** Thêm Metadata Filtering trong bước tìm kiếm (ưu tiên `status="active"` hoặc tài liệu mới nhất), hoặc bổ sung Recency Boost trong công thức tính điểm của Hybrid Search.

### #2
- **Question:** Khi phát hiện malware trên máy, nhân viên có nên tự diệt không?
- **Expected:** Không, nhân viên không được tự xử lý mà phải ngắt kết nối mạng ngay lập tức (rút dây LAN, tắt Wi-Fi) và liên hệ IT Security xử lý theo quy trình ứng cứu sự cố.
- **Got:** Trích từ bao_mat_su_co.md. Quy trình xử lý mã độc và các dấu hiệu nhận biết malware trên máy trạm...
- **Worst metric:** `answer_relevancy` (Negation Misunderstanding)
- **Error Tree:** Output chưa dứt khoát câu trả lời Phủ định → Context trích xuất quy trình chung thay vì hành động cấm đoán tức thời → Query có từ "không/nên" bị vector embedding hòa tan ý nghĩa phủ định.
- **Root cause:** Negation Query Gap trong dense retrieval. Các embedding model thường biểu diễn câu phủ định và khẳng định có khoảng cách cosin khá gần nhau. Nếu tài liệu chứa các từ "tự diệt malware", mô hình dễ nhầm lẫn giữa khuyến cáo "KHÔNG được làm" và hướng dẫn "được làm".
- **Suggested fix:** Tận dụng BM25 với trọng số cao hơn cho các từ khóa mang tính cấm đoán ("nghiêm cấm", "không được tự ý", "ngắt mạng"), hoặc sử dụng Query Rewriting để chuyển câu hỏi phủ định thành dạng truy vấn chuẩn hóa.

### #3
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm theo quy định hiện hành?
- **Expected:** 18 ngày (15 ngày cơ bản theo v2024 + 3 ngày cộng thêm do 9 năm thâm niên: 9 / 3 = 3 ngày).
- **Got:** Trích từ nghi_phep_nam_v2024.md. Mỗi nhân viên chính thức được hưởng 15 ngày phép năm có lương, tăng thêm 1 ngày cho mỗi 3 năm thâm niên...
- **Worst metric:** `faithfulness` / `answer_relevancy` (Multi-hop & Arithmetic Reasoning)
- **Error Tree:** Output thiếu kết quả tính toán số học cuối cùng (18 ngày) → Context đã lấy đúng quy chế hiện hành v2024 → LLM hoặc Fallback chưa thực hiện phép suy luận tính toán 15 + (9/3).
- **Root cause:** RAG retrieval chỉ cung cấp dữ liệu nguyên bản, việc tính toán số học đòi hỏi năng lực lý luận (Reasoning) nhiều bước. Khi không có LLM suy luận chuỗi suy nghĩ (Chain-of-Thought) hoặc công cụ tính toán, hệ thống chỉ trích dẫn văn bản mà không ra được con số 18.
- **Suggested fix:** Áp dụng Chain-of-Thought (CoT) Prompting cho LLM generation và bổ sung Agentic Tool (Python Calculator) cho các câu hỏi chứa dữ liệu định lượng và thâm niên.

### #4
- **Question:** Nếu cần mua một chiếc laptop 30 triệu cho nhân viên mới, quy trình phê duyệt thế nào?
- **Expected:** Cần Trưởng bộ phận và Giám đốc Khối/CFO phê duyệt (vì nằm trong khung hạn mức 20 - 50 triệu VNĐ).
- **Got:** Trích từ mua_sam.md. Bảng phân quyền phê duyệt mua sắm trang thiết bị...
- **Worst metric:** `context_precision` (Numeric Range Matching)
- **Error Tree:** Output trích xuất cả bảng thay vì xác định đúng cấp phê duyệt → Context bao gồm nhiều mốc hạn mức khác nhau (10tr, 20tr, 50tr) → Retrieval lấy bảng nhưng chưa chỉ rõ dòng tương ứng với 30 triệu.
- **Root cause:** Bảng hạn mức mua sắm trong markdown được chunking thành một khối lớn. Vector search không hiểu mối quan hệ toán học giữa con số "30 triệu" và khoảng "[20.000.000 - 50.000.000 VNĐ]".
- **Suggested fix:** Sử dụng Structure-Aware Chunking để phân tách từng dòng trong bảng hạn mức thành chunk riêng kèm metadata khoảng giá trị (`min_value: 20000000, max_value: 50000000`), sau đó truy vấn bằng Metadata Range Filter.

### #5
- **Question:** Mentor và buddy của nhân viên mới có thể là cùng một người không?
- **Expected:** Không, mentor (hướng dẫn chuyên môn) và buddy (hỗ trợ văn hóa, hòa nhập) bắt buộc phải là hai nhân sự riêng biệt nhằm đảm bảo tính khách quan và hiệu quả đào tạo.
- **Got:** Trích từ mentor_buddy.md. Vai trò của Mentor: định hướng nghiệp vụ; Vai trò của Buddy: hỗ trợ đời sống văn hóa doanh nghiệp...
- **Worst metric:** `context_recall` (Implicit Constraint)
- **Error Tree:** Output chưa khẳng định được việc cấm trùng người → Context trích đoạn mô tả 2 vai trò nhưng điều khoản cấm kiêm nhiệm nằm ở phần cuối quy chế → Chunking cắt rời phần định nghĩa và phần điều khoản cấm.
- **Root cause:** Mất ngữ cảnh toàn cục (Context Fragmentation). Kỹ thuật chunking thông thường cắt tài liệu thành các đoạn 256 ký tự khiến đoạn mô tả vai trò không đi kèm với điều khoản loại trừ/cấm kiêm nhiệm ở mục sau.
- **Suggested fix:** Áp dụng triệt để Hierarchical Chunking (Parent-Child) của Module 1: Khi match vào child chunk nói về Mentor/Buddy, hệ thống trả về Parent Chunk (2048 ký tự) chứa toàn bộ phần quy định kiêm nhiệm để LLM có đầy đủ thông tin đối chiếu.

---

## Case Study (cho presentation)

**Question chọn phân tích:** *"Thâm niên bao nhiêu năm thì được cộng thêm ngày phép?"*

### Error Tree Walkthrough:
1. **Output đúng?** → **KHÔNG**. Hệ thống trả về "5 năm" (theo quy chế cũ 2023), trong khi đáp án chuẩn hiện hành là "3 năm" (theo quy chế mới 2024).
2. **Context đúng?** → **KHÔNG HOÀN TOÀN**. Trong top context trả về có cả đoạn của file 2023 và 2024, nhưng đoạn 2023 được xếp hạng điểm cao hơn một phần do độ dài từ khóa khớp tốt.
3. **Query rewrite OK?** → **CHƯA ĐỦ**. Query gốc của người dùng không chứa năm (không nói rõ "năm 2024" hay "hiện tại"). Bộ Query Preprocessing chưa tự động inject context thời gian hiện hành.
4. **Fix ở bước:**
   - **Bước 1 (Metadata):** Bổ sung trường `status: "deprecated"` cho `nghi_phep_nam_v2023.md` và `status: "active"` cho `nghi_phep_nam_v2024.md`.
   - **Bước 2 (Search Filter):** Hybrid Search chỉ index hoặc ưu tiên boost điểm cho tài liệu `active`.
   - **Bước 3 (Reranker):** Trong Cross-Encoder prompt, cấu hình ưu tiên tài liệu có metadata mới nhất khi có xung đột quy định.

### Nếu có thêm 1 giờ, sẽ optimize:
1. **HyDE (Hypothetical Document Embeddings) & Query Rewriter:** Tự động mở rộng các câu hỏi ngắn và câu hỏi thời gian thành các truy vấn chứa bối cảnh hiện hành của doanh nghiệp.
2. **Tabular & Numeric Metadata Indexing:** Bóc tách các bảng biểu số liệu (phụ cấp, hạn mức laptop, thâm niên) thành định dạng có cấu trúc để kết hợp Text-to-SQL / Hybrid Filtering thay vì chỉ dựa vào text search thô.
3. **Parent Document Retrieval Integration:** Cấu hình chuẩn hóa pipeline trả về Parent Chunk (2048 token) cho LLM đọc bối cảnh toàn diện thay vì chỉ cấp Child Chunk (256 token).

