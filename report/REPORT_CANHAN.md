# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Phạm Đức Anh

**MSSV:** 2A202602994

**Nhóm:** Top1Server

**Ngày:** 19/09/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm về bộ tài liệu, chiến lược chung, benchmark và demo được trình bày trong `REPORT_NHOM.md`.

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity)

**Độ tương tự cosine cao nghĩa là gì?**

Độ tương tự cosine cao nghĩa là hai vector embedding có hướng gần nhau, từ đó cho thấy hai đoạn văn có nội dung hoặc ý nghĩa tương đồng. Giá trị gần 1 thể hiện mức tương đồng cao, gần 0 là ít liên quan và gần -1 là có hướng đối lập.

**Ví dụ có độ tương tự cao:**

- Câu A: “Sinh viên đăng ký học phần trên cổng học vụ.”
- Câu B: “Người học ghi danh môn học qua hệ thống đào tạo.”
- Hai câu dùng từ khác nhau nhưng cùng diễn đạt hành động đăng ký môn học.

**Ví dụ có độ tương tự thấp:**

- Câu A: “Python là một ngôn ngữ lập trình.”
- Câu B: “Hôm nay trời mưa rất lớn.”
- Hai câu thuộc hai chủ đề và ngữ cảnh hoàn toàn khác nhau.

**Tại sao ưu tiên cosine similarity hơn Euclidean distance cho text embeddings?**

Cosine similarity tập trung vào hướng của vector, tức quan hệ ngữ nghĩa, và ít bị ảnh hưởng bởi độ lớn của vector. Khoảng cách Euclid phụ thuộc nhiều hơn vào độ lớn nên hai vector cùng hướng nhưng khác độ dài vẫn có thể bị xem là xa nhau.

### Bài toán Chunking

Với tài liệu 10.000 ký tự, `chunk_size=500`, `overlap=50`:

```text
ceil((10000 - 50) / (500 - 50))
= ceil(9950 / 450)
= ceil(22.111...)
= 23 chunks
```

Nếu tăng overlap lên 100:

```text
ceil((10000 - 100) / (500 - 100))
= ceil(9900 / 400)
= ceil(24.75)
= 25 chunks
```

Số chunk tăng từ 23 lên 25 vì bước dịch cửa sổ giảm từ 450 xuống 400 ký tự. Overlap lớn hơn giúp giữ lại ngữ cảnh ở ranh giới giữa hai chunk, nhưng làm tăng số vector cần lưu trữ và tìm kiếm.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

### Các hàm Chunking

**`SentenceChunker.chunk`**

Tôi dùng regex `(?<=[.!?])\s+` để tách tại khoảng trắng đứng sau dấu kết thúc câu, nhờ đó dấu câu không bị mất. Sau khi loại khoảng trắng thừa, các câu được gom theo nhóm có tối đa `max_sentences_per_chunk` câu. Text rỗng trả về danh sách rỗng; hạn chế hiện tại là chữ viết tắt như `TS.` và số thập phân có thể bị nhận diện sai thành ranh giới câu.

**`RecursiveChunker.chunk` và `_split`**

Thuật toán thử các separator từ ranh giới lớn đến nhỏ: đoạn văn, dòng, câu, từ rồi cắt cứng. Mảnh còn quá dài được xử lý đệ quy bằng separator tiếp theo, còn các mảnh nhỏ liên tiếp được gom lại tới gần `chunk_size`. Base case là text đã đủ ngắn, danh sách separator rỗng hoặc separator cuối là chuỗi rỗng; khi đó text được cắt cứng theo kích thước yêu cầu.

**`compute_similarity` và `ChunkingStrategyComparator`**

Cosine similarity được tính bằng tích vô hướng chia cho tích độ lớn hai vector; nếu một vector có độ lớn bằng 0 thì hàm trả `0.0`. Comparator chạy `FixedSizeChunker`, `SentenceChunker` và `RecursiveChunker` trên cùng văn bản, sau đó trả số chunk, độ dài trung bình và nội dung chunk của từng chiến lược. Trường hợp text rỗng được xử lý để không chia cho 0.

### Lớp EmbeddingStore

**`add_documents` và `search`**

Mỗi `Document` được chuẩn hóa thành record gồm id, content, bản sao metadata và embedding. Store dùng danh sách trong bộ nhớ; khi tìm kiếm, query được embedding một lần, tính dot product với từng record rồi sắp xếp score giảm dần. Vì các embedding đã được chuẩn hóa, dot product tương đương cosine similarity.

**`search_with_filter` và `delete_document`**

Metadata được lọc trước khi similarity search để các vị trí top-k không bị tài liệu sai đối tượng chiếm mất. Mỗi record luôn có `metadata['doc_id']`; `delete_document` xóa toàn bộ chunk có cùng `doc_id` và trả `True` nếu số record thực sự giảm.

### Tác tử KnowledgeBaseAgent

**`answer`**

Agent thực hiện ba bước: truy xuất top-k chunk, ghép chúng thành context có đánh số nguồn `[1]`, `[2]`, ... rồi gọi `llm_fn`. Prompt yêu cầu chỉ trả lời bằng ngữ cảnh được cung cấp và nói rõ khi thiếu thông tin để hạn chế hallucination. Nếu store không có kết quả, agent trả thông báo ngay và không gọi LLM không cần thiết.

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

### Kết quả kiểm thử

```text
Ran 42 tests in 0.005s

OK
```

Các nhóm test đã vượt qua gồm project structure, ba chunker, cosine similarity, comparator, vector store, metadata filtering, delete document và KnowledgeBaseAgent.

**Số lượng bài test vượt qua:** **42 / 42**

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

Backend dùng để đo: `MockEmbedder(dim=64)`. Đây là embedding xác định dựa trên hash, phục vụ unit test chứ không biểu diễn ngữ nghĩa thực sự.

| Cặp | Câu A | Câu B | Dự đoán | Điểm thực tế | Đúng? |
|---|---|---|---|---:|---|
| 1 | Sinh viên đăng ký học phần trên cổng học vụ. | Người học ghi danh môn học qua hệ thống đào tạo. | Cao | 0.232875 | Đúng tương đối — cao nhất trong 5 cặp |
| 2 | Thư viện cho sinh viên mượn sách. | Sinh viên có thể gia hạn tài liệu tại thư viện. | Cao | -0.029033 | Không |
| 3 | Python là một ngôn ngữ lập trình. | Hôm nay trời mưa rất lớn. | Thấp | 0.118700 | Đúng về dự đoán ngữ nghĩa, nhưng mock cho score dương |
| 4 | Sinh viên phải đóng học phí đúng hạn. | Người học cần thanh toán tuition trước thời hạn. | Cao | -0.083449 | Không |
| 5 | Con chó đang chạy trong công viên. | Máy tính lượng tử sử dụng qubit. | Thấp | 0.015116 | Đúng |

**Kết quả bất ngờ và reflection**

Cặp 2 và cặp 4 tương đồng rõ về ngữ nghĩa nhưng lại nhận score âm. Nguyên nhân là `MockEmbedder` sinh vector từ hash của chuỗi chứ không hiểu nghĩa, vì vậy số đo chỉ có tính xác định để kiểm thử code. Khi benchmark retrieval thật, nhóm nên dùng embedding đa ngữ như `paraphrase-multilingual-MiniLM-L12-v2`, OpenAI hoặc Gemini thay vì diễn giải score của mock như chất lượng ngữ nghĩa.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Chiến lược được phân công của tôi là `RecursiveChunker(chunk_size=500)`. Tôi dùng chung 8 tài liệu, 5 query, gold answer, `top_k=3` và embedding đa ngữ `paraphrase-multilingual-MiniLM-L12-v2` với cả nhóm để việc so sánh công bằng. Quy tắc chấm là 2 điểm nếu context top-3 chứa đủ các dữ kiện bắt buộc, 1 điểm nếu chỉ chứa một phần và 0 điểm nếu không đủ thông tin trả lời.

| # | Câu hỏi (Query) | Kết quả RecursiveChunker | Điểm | Nhận xét |
|---|---|---|---:|---|
| 1 | Sinh viên được mượn tối đa bao nhiêu tài liệu và trong bao nhiêu ngày? | Context có đúng bảng “Sinh viên, học viên — 3 — 10”; dùng `metadata_filter={"audience": "student"}` | 2 / 2 | Đây là câu bẫy metadata vì tài liệu giảng viên có hạn mức khác |
| 2 | Điều kiện nào để được mượn tài liệu về nhà? | Truy xuất đúng tài liệu nhưng chunk chưa giữ đủ dữ kiện 25/35, thẻ và tiền cược trong cùng context | 0 / 2 | Failure case do ranh giới chunk tách mất dữ kiện cần thiết |
| 3 | Quy trình đăng ký phòng học nhóm và quy định đến trễ? | Context giữ được quy trình đặt phòng và mốc hủy sau 15 phút | 2 / 2 | Recursive split phù hợp với danh sách nhiều bước |
| 4 | Hạn mức, thời hạn và phí trễ mượn liên thư viện? | Context chứa đủ 2 tài liệu, 20 ngày và 5.000 đồng/tài liệu/ngày | 2 / 2 | Các con số nằm gần nhau trong một mục ngắn |
| 5 | Lệ phí và thời gian cấp thẻ thư viện? | Context chứa đủ 100.000 đồng/thẻ và 07 ngày kể từ ngày đăng ký | 2 / 2 | Bảng phí và thời gian vẫn nằm trong top-3 context |

**Kết quả theo bảng tổng hợp nhóm:** **8 / 10**. `RecursiveChunker` là chiến lược có tổng điểm cao nhất trong bốn chiến lược của nhóm trên bộ query hiện tại.

**Lưu ý về khả năng tái lập:** Bảng nhóm ghi lần chạy LocalEmbedder tạo 83 chunk, trong khi phiên bản code hiện tại tạo 76 chunk khi kiểm tra cấu trúc. Vì số chunk không phụ thuộc embedding backend, nhóm cần chạy lại lệnh `python bench.py --strategy recursive` và lưu nguyên output vào `ket_qua_benchmark_recursive.txt` trước khi nộp để xác nhận số liệu cuối cùng. Tôi không dùng kết quả MockEmbedder 0/10 để đánh giá chất lượng semantic vì backend này chỉ phục vụ unit test.

**Phân tích failure case:** Câu 2 cho thấy truy xuất đúng `doc_id` chưa đủ để coi là trả lời đúng. Các điều kiện “25/35”, thẻ thư viện và tiền cược phải cùng xuất hiện trong context; RecursiveChunker có thể tách chúng qua ranh giới đoạn. Cách cải thiện là gắn heading cha vào mỗi chunk con, thêm overlap có kiểm soát hoặc dùng chiến lược hybrid heading + recursive.

**Điều học được từ nhóm:** `HeadingChunker` tốt ở câu tra quy định theo mục, còn overlap của `FixedSizeChunker` hữu ích khi dữ kiện nằm sát ranh giới. Nếu làm lại, tôi vẫn dùng recursive làm fallback nhưng sẽ giữ heading hiện tại trong metadata và nội dung từng chunk, sau đó đo lại trên cùng LocalEmbedder thay vì chọn chiến lược chỉ dựa vào trực giác.

---

## Tự đánh giá hiện tại

| Tiêu chí | Điểm tự đánh giá |
|---|---:|
| Khởi động | 5 / 5 |
| Hướng tiếp cận | 10 / 10 |
| Hoàn thiện code | 30 / 30 |
| Dự đoán độ tương tự | 5 / 5 |
| Kết quả truy xuất cá nhân | 8 / 10 theo bảng tổng hợp nhóm; cần lưu raw output Recursive để khóa bằng chứng |
| **Tổng hiện tại** | **58 / 60** |

## Checklist cá nhân trước khi nộp

- [x] Điền đầy đủ thông tin cá nhân, nhóm và vai trò R2 Benchmark.
- [x] Giải thích cosine similarity, chunking và ba thành phần code chính.
- [x] Chạy và ghi nhận 42/42 unit tests.
- [x] Chuẩn bị 5 query, gold answer và đánh dấu câu bẫy metadata.
- [x] Phân tích ít nhất một failure case và đề xuất cải tiến.
- [ ] Chạy lại LocalEmbedder cho `RecursiveChunker` trên code hiện tại và lưu `ket_qua_benchmark_recursive.txt`.
- [ ] Đối chiếu lại số chunk 76/83 rồi thống nhất con số ở cả báo cáo cá nhân và báo cáo nhóm.
