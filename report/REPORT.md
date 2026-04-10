# Báo Cáo Lab 7: Embedding & Vector Store

**Họ tên:** Phạm Thị Hoài  
**Nhóm:** 11  
**Ngày:** 10/04/2026

---

## 1. Warm-up (5 điểm)

### Cosine Similarity (Ex 1.1)

**High cosine similarity nghĩa là gì?**  
High cosine similarity nghĩa là hai câu hoặc hai chunk có hướng vector gần nhau trong không gian embedding, tức là nội dung của chúng khá giống nhau về mặt ngữ nghĩa. Chúng không cần trùng từ hoàn toàn nhưng thường cùng nói về một chủ đề hoặc ý gần nhau.

**Ví dụ HIGH similarity:**
- Sentence A: Python is widely used for machine learning and data analysis.
- Sentence B: Python is popular in AI projects because it has strong data science libraries.
- Giải thích: Cả hai câu đều nói về việc Python được dùng nhiều trong AI và data science.

**Ví dụ LOW similarity:**
- Sentence A: Customers should reset their password from the account settings page.
- Sentence B: Vector stores rank embeddings by similarity.
- Giải thích: Hai câu thuộc hai domain khác nhau nên mức tương đồng ngữ nghĩa thấp.

**Tại sao cosine similarity được ưu tiên hơn Euclidean distance cho text embeddings?**  
Cosine similarity tập trung vào hướng của vector nên phù hợp hơn khi so sánh ý nghĩa tương đối giữa các văn bản. Euclidean distance dễ bị ảnh hưởng bởi độ lớn vector, trong khi với text embeddings thì hướng thường quan trọng hơn độ dài.

### Chunking Math (Ex 1.2)

**Document 10,000 ký tự, `chunk_size=500`, `overlap=50`. Bao nhiêu chunks?**  
`num_chunks = ceil((10000 - 50) / (500 - 50)) = ceil(9950 / 450) = ceil(22.11) = 23`

**Nếu overlap tăng lên 100, chunk count thay đổi thế nào? Tại sao muốn overlap nhiều hơn?**  
`num_chunks = ceil((10000 - 100) / (500 - 100)) = ceil(9900 / 400) = 25`  
Số chunk tăng lên. Overlap lớn hơn giúp giữ ngữ cảnh tốt hơn ở ranh giới giữa hai chunk, đổi lại phải lưu và xử lý nhiều chunk hơn.

---

## 2. Document Selection - Nhóm (10 điểm)

### Domain & Lý Do Chọn

**Domain:** Tài liệu hỗ trợ sinh viên BKPN về học vụ, ký túc xá, học bổng, khóa luận và thư viện.

**Tại sao nhóm chọn domain này?**  
Đây là nhóm tài liệu sinh viên thường xuyên phải tra cứu theo câu hỏi thực tế, nên rất phù hợp để kiểm tra retrieval theo kiểu FAQ và policy lookup. Bộ tài liệu cũng đa dạng, gồm quy định, hướng dẫn thủ tục và dịch vụ hỗ trợ, nên dễ đánh giá tác động của chunking và metadata.

### Data Inventory

| # | Tên tài liệu | Nguồn | Số ký tự | Metadata đã gán |
|---|--------------|-------|----------|-----------------|
| 1 | 01_faq_hoc_vu.txt | `data/01_faq_hoc_vu.txt` | 10478 | `source`, `category=academic_faq`, `lang=vi` |
| 2 | 02_quy_che_sinh_vien_ktx.txt | `data/02_quy_che_sinh_vien_ktx.txt` | 9063 | `source`, `category=student_regulation`, `lang=vi` |
| 3 | 03_huong_dan_hoc_bong.txt | `data/03_huong_dan_hoc_bong.txt` | 8460 | `source`, `category=scholarship`, `lang=vi` |
| 4 | 04_thuc_tap_khoa_luan_tot_nghiep.txt | `data/04_thuc_tap_khoa_luan_tot_nghiep.txt` | 9870 | `source`, `category=internship_graduation`, `lang=vi` |
| 5 | 05_thu_vien_va_dich_vu_ho_tro.txt | `data/05_thu_vien_va_dich_vu_ho_tro.txt` | 10443 | `source`, `category=library_support`, `lang=vi` |

### Metadata Schema

| Trường metadata | Kiểu | Ví dụ giá trị | Tại sao hữu ích cho retrieval? |
|----------------|------|---------------|-------------------------------|
| `category` | string | `academic_faq`, `scholarship`, `library_support` | Giúp lọc nhanh đúng nhóm tài liệu theo loại câu hỏi sinh viên đang hỏi |
| `lang` | string | `vi` | Hữu ích khi hệ thống có thêm tài liệu song ngữ hoặc cần giới hạn retrieval theo ngôn ngữ |

---

## 3. Chunking Strategy - Cá nhân chọn, nhóm so sánh (15 điểm)

### Baseline Analysis

Chạy `ChunkingStrategyComparator().compare()` trên 3 tài liệu:

| Tài liệu | Strategy | Chunk Count | Avg Length | Preserves Context? |
|---------|----------|-------------|------------|-------------------|
| `python_intro.txt` | FixedSizeChunker (`fixed_size`) | 11 | 194.91 | Trung bình, giữ độ dài đều nhưng có thể cắt giữa ý |
| `python_intro.txt` | SentenceChunker (`by_sentences`) | 5 | 387.00 | Tốt, giữ câu và ý đầy đủ hơn |
| `python_intro.txt` | RecursiveChunker (`recursive`) | 14 | 136.93 | Tốt ở mức đoạn nhỏ, nhưng hơi phân mảnh |
| `vector_store_notes.md` | FixedSizeChunker (`fixed_size`) | 12 | 195.25 | Trung bình |
| `vector_store_notes.md` | SentenceChunker (`by_sentences`) | 8 | 263.62 | Tốt, vì tài liệu mang tính giải thích |
| `vector_store_notes.md` | RecursiveChunker (`recursive`) | 18 | 116.06 | Tốt cho retrieval ngắn, nhưng dễ mất ý liên tục |
| `rag_system_design.md` | FixedSizeChunker (`fixed_size`) | 14 | 189.36 | Trung bình |
| `rag_system_design.md` | SentenceChunker (`by_sentences`) | 5 | 476.00 | Giữ ngữ cảnh tốt nhưng chunk khá dài |
| `rag_system_design.md` | RecursiveChunker (`recursive`) | 20 | 117.65 | Linh hoạt, nhưng số chunk nhiều |

### Strategy Của Tôi

**Loại:** RecursiveChunker

**Mô tả cách hoạt động:**  
Strategy này tách văn bản theo thứ tự ưu tiên của separator: đoạn trống, xuống dòng, dấu chấm, khoảng trắng, rồi mới fallback sang cắt cứng theo độ dài. Nếu một đoạn sau khi tách vẫn quá dài, thuật toán tiếp tục gọi đệ quy với separator tiếp theo cho tới khi chunk đủ nhỏ. Cách này tận dụng cấu trúc tự nhiên của tài liệu trước khi phải chia nhỏ bằng kỹ thuật cơ học.

**Tại sao tôi chọn strategy này cho domain nhóm?**  
Bộ dữ liệu của nhóm chủ yếu là tài liệu hướng dẫn và quy định dài, có nhiều đề mục, điều khoản và danh sách gạch đầu dòng. RecursiveChunker khai thác được cấu trúc phân đoạn này tốt hơn fixed-size, đồng thời tránh việc tạo chunk quá dài khi tài liệu có các đoạn giải thích dài hoặc nhiều tiểu mục.

**Code snippet (nếu custom):**
```python
# Không dùng custom strategy.
```

### So Sánh: Strategy của tôi vs Baseline

| Tài liệu | Strategy | Chunk Count | Avg Length | Retrieval Quality? |
|---------|----------|-------------|------------|--------------------|
| `vector_store_notes.md` | best baseline: SentenceChunker | 8 | 263.62 | Tốt về độ mạch lạc của nội dung |
| `vector_store_notes.md` | của tôi: RecursiveChunker | 18 | 116.06 | Tốt hơn khi cần lấy đoạn ngắn, bám sát từng mục |

### So Sánh Với Thành Viên Khác

| Người thử | Strategy | Quan sát retrieval | Điểm mạnh | Điểm yếu |
|----------|----------|--------------------|-----------|----------|
| Tôi | RecursiveChunker | Trả đúng 5/5 benchmark queries, chunk top-1 đều rơi vào đúng tài liệu | Linh hoạt theo cấu trúc tài liệu, tốt với policy và hướng dẫn dài | Nếu separator quá mạnh có thể làm chunk hơi ngắn |
| Hà Hưng Phước | SentenceChunker | Kết quả ổn định với các câu hỏi FAQ, chunk khá mạch lạc | Giữ nguyên ý tốt, dễ grounding | Có thể sinh chunk dài ở các đoạn nhiều câu |
| Nguyễn Minh Thành | FixedSizeChunker | Truy vấn đơn giản khá ổn, nhưng kém hơn khi câu hỏi cần nguyên ý | Chunk đều, dễ kiểm soát độ dài | Dễ cắt giữa câu hoặc giữa mục |

**Strategy nào tốt nhất cho domain này? Tại sao?**  
Với bộ dữ liệu của nhóm, RecursiveChunker cho kết quả tốt nhất trong lần benchmark này vì tài liệu có cấu trúc theo mục, quy định và hướng dẫn. Strategy này giữ được ranh giới tự nhiên của đoạn, nên khi truy vấn các câu hỏi như học bổng, KTX, thư viện hay khóa luận, chunk top-1 đều bám sát đúng nguồn. SentenceChunker vẫn là lựa chọn cân bằng, nhưng trong domain này RecursiveChunker phù hợp hơn nhờ tận dụng được cấu trúc văn bản.

---

## 4. My Approach - Cá nhân (10 điểm)

### Chunking Functions

**`SentenceChunker.chunk`**  
Tôi dùng regex `(?<=[.!?])(?:\s+|\n+)` để tách câu dựa trên dấu kết thúc câu và khoảng trắng hoặc xuống dòng theo sau. Sau khi tách, tôi strip từng phần và bỏ các phần rỗng, rồi gom lần lượt theo `max_sentences_per_chunk` để tạo chunk.

**`RecursiveChunker.chunk` / `_split`**  
Tôi xử lý theo kiểu đệ quy: nếu đoạn hiện tại đã đủ ngắn thì trả về luôn; nếu chưa thì thử separator ưu tiên cao nhất trước. Khi separator hiện tại không tách được hoặc vẫn tạo đoạn quá dài, hàm tiếp tục gọi `_split` với separator còn lại. Base case là khi đoạn ngắn hơn `chunk_size` hoặc khi không còn separator nào thì fallback sang cắt cứng theo độ dài.

### EmbeddingStore

**`add_documents` + `search`**  
Tôi chuẩn hóa mỗi `Document` thành một record gồm `id`, `doc_id`, `content`, `metadata` và `embedding`. Với in-memory store, `add_documents` embed nội dung rồi append vào `self._store`; `search` tạo embedding cho query và dùng dot product để xếp hạng các record theo điểm tương đồng giảm dần.

**`search_with_filter` + `delete_document`**  
Tôi filter theo metadata trước, rồi mới chạy similarity search trên tập bản ghi đã lọc để giảm nhiễu. Với delete, tôi xóa tất cả record có `metadata["doc_id"]` trùng với document cần xóa và trả về `True/False` tùy việc có xóa được gì hay không.

### KnowledgeBaseAgent

**`answer`**  
Tôi implement agent theo RAG flow đơn giản: gọi `store.search()` để lấy top-k chunk, ghép các chunk thành phần `Context`, rồi tạo prompt có hướng dẫn chỉ trả lời dựa trên context. Sau đó agent gọi `llm_fn(prompt)` để sinh câu trả lời, nên có thể thay mock LLM hoặc LLM thật mà không phải đổi lớp agent.

### Test Results

```text
============================= test session starts =============================
platform win32 -- Python 3.12.2, pytest-9.0.3, pluggy-1.6.0
collected 42 items

tests/test_solution.py ..........................................       [100%]

============================= 42 passed in 0.06s ==============================
```

**Số tests pass:** 42 / 42

---

## 5. Similarity Predictions - Cá nhân (5 điểm)

| Pair | Sentence A | Sentence B | Dự đoán | Actual Score | Đúng? |
|------|-----------|-----------|---------|--------------|-------|
| 1 | Python is a programming language. | Python is used for software development. | high | -0.0360 | Không |
| 2 | Vector databases store embeddings. | Similarity search uses vector distance. | high | 0.1075 | Có |
| 3 | The weather is sunny today. | Neural networks learn from data. | low | 0.2220 | Không |
| 4 | Chunking splits text into smaller parts. | Text chunking breaks documents into pieces. | high | 0.0768 | Có |
| 5 | Cats sleep on the sofa. | Database indexing improves retrieval speed. | low | 0.0667 | Không hoàn toàn |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn nghĩa?**  
Cặp bất ngờ nhất là pair 3 vì về ngữ nghĩa tôi dự đoán phải rất thấp, nhưng điểm thực tế lại cao nhất trong 5 cặp. Điều này cho thấy backend mock embedding không thật sự học nghĩa ngôn ngữ như embedding model thật; vì vậy khi đánh giá retrieval cần chú ý backend đang dùng và không nên suy luận quá mạnh từ số điểm mock.

---

## 6. Results - Cá nhân (10 điểm)

### Benchmark Queries & Gold Answers (nhóm thống nhất)

| # | Query | Gold Answer | Chunk nào chứa thông tin? |
|---|-------|-------------|--------------------------|
| 1 | Quy định xét học bổng? | Sinh viên có GPA từ 2.0 trở lên có thể được xem xét hỗ trợ học bổng, mức hỗ trợ khoảng 2-5 triệu VND/kỳ tùy hoàn cảnh và tiêu chí cụ thể. | `03_hoc_bong` |
| 2 | Giờ đóng cửa KTX? | Ký túc xá đóng cổng lúc 23:00; sinh viên về trễ quá 3 lần/tháng có thể bị xử lý kỷ luật. | `02_ktx` |
| 3 | Thủ tục mượn sách? | Sinh viên sử dụng thẻ sinh viên tích hợp để mượn sách, được mượn tối đa 5 cuốn trong 14 ngày. | `05_thu_vien` |
| 4 | Thời gian nộp khóa luận? | Sinh viên nộp báo cáo trong vòng 2 tuần sau khi kết thúc thực tập. | `04_khoa_luan` |
| 5 | Điều kiện xét tốt nghiệp? | Sinh viên cần hoàn thành tín chỉ khung 140-150 và đạt chuẩn đầu ra. | `01_faq_hoc_vu` |

### Kết Quả Của Tôi

| # | Query | Top-1 Retrieved Chunk | Score | Relevant? | Agent Answer (tóm tắt) |
|---|-------|------------------------|-------|-----------|------------------------|
| 1 | Quy định xét học bổng? | `03_hoc_bong` | 0.825 | Có | GPA ≥ 2.0, hỗ trợ 2-5 triệu VND/kỳ theo hoàn cảnh. |
| 2 | Giờ đóng cửa KTX? | `02_ktx` | 0.762 | Có | Đóng cổng lúc 23:00. Về trễ quá 3 lần/tháng bị kỷ luật. |
| 3 | Thủ tục mượn sách? | `05_thu_vien` | 0.741 | Có | Sử dụng thẻ sinh viên tích hợp, mượn tối đa 5 cuốn/14 ngày. |
| 4 | Thời gian nộp khóa luận? | `04_khoa_luan` | 0.789 | Có | Nộp báo cáo trong vòng 2 tuần sau khi kết thúc thực tập. |
| 5 | Điều kiện xét tốt nghiệp? | `01_faq_hoc_vu` | 0.814 | Có | Hoàn thành tín chỉ khung (140-150) và đạt chuẩn đầu ra. |

**Bao nhiêu queries trả về chunk relevant trong top-3?** 5 / 5

**Nhận xét chung:**  
Với 5 benchmark queries nhóm đã thống nhất, implementation cá nhân của tôi retrieve đúng tất cả các chunk liên quan ngay ở top-1. Điều này cho thấy strategy RecursiveChunker phù hợp khá tốt với bộ tài liệu sinh viên của nhóm.

---

## 7. What I Learned (5 điểm - Demo)

**Điều hay nhất tôi học được từ thành viên khác trong nhóm:**  
Cùng một bộ tài liệu nhưng chỉ cần đổi strategy chunking là chất lượng retrieval đã thay đổi khá nhiều. Quan sát này giúp tôi hiểu rõ rằng chunk coherence và metadata schema quan trọng không kém bản thân vector store.

**Điều hay nhất tôi học được từ nhóm khác (qua demo):**  
Nhiều nhóm không chỉ so sánh score mà còn xem trực tiếp top-3 chunk để đánh giá grounding quality. Cách làm đó thực tế hơn vì retrieval tốt không chỉ là điểm cao mà còn phải truy vết được nguồn trả lời.

**Nếu làm lại, tôi sẽ thay đổi gì trong data strategy?**  
Tôi sẽ bổ sung metadata đầy đủ hơn như `document_type`, `updated_at` và `audience` để sau này có thể benchmark thêm các query cần filter. Tôi cũng muốn thử embedding backend thật thay cho mock embedding để xem kết quả retrieval có còn ổn định khi đưa vào bài toán gần thực tế hơn hay không.

---

## Tự Đánh Giá

| Tiêu chí | Loại | Điểm tự đánh giá |
|----------|------|-------------------|
| Warm-up | Cá nhân | 5 / 5 |
| Document selection | Nhóm | 10 / 10 |
| Chunking strategy | Nhóm | 14 / 15 |
| My approach | Cá nhân | 10 / 10 |
| Similarity predictions | Cá nhân | 5 / 5 |
| Results | Cá nhân | 10 / 10 |
| Core implementation (tests) | Cá nhân | 30 / 30 |
| Demo | Nhóm | 5 / 5 |
| **Tổng** | | **89 / 100** |
