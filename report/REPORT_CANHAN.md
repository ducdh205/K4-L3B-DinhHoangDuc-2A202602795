# Báo Cáo Cá Nhân — Lab 7: Embedding & Vector Store

**Họ tên:** Đinh Hoàng Đức — 2A202602795
**Nhóm:** T153
**Ngày:** 20/09/2026

> **Nộp 1 bản / sinh viên.** Phần nhóm (lựa chọn tài liệu, thiết kế chiến lược, bộ câu hỏi đánh giá, demo) nộp chung 1 bản trong `REPORT_NHOM.md`. Chi tiết thang điểm: `docs/SCORING.md`.

**Tổng điểm phần cá nhân: 60** = Khởi động (5) + Hướng tiếp cận (10) + Hoàn thiện code (30) + Dự đoán độ tương tự (5) + Kết quả truy xuất của tôi (10).

---

## 1. Khởi động (Warm-up) — Cá nhân (5 điểm)

### Độ tương tự Cosine (Cosine Similarity) (Bài tập 1.1)

**Độ tương tự cosine cao (High cosine similarity) nghĩa là gì?**

Độ tương tự cosine cao nghĩa là hai vector có hướng gần nhau, nên hai văn bản thường có nội dung hoặc ngữ nghĩa liên quan. Điểm gần `1` biểu thị rất tương đồng; điểm gần `0` biểu thị ít liên hệ theo cách biểu diễn vector đó.

**Ví dụ có độ tương tự CAO:**

- Câu A: Người mua có thể yêu cầu hoàn tiền cho đơn hàng đã giao.
- Câu B: Tôi muốn gửi yêu cầu Trả hàng/Hoàn tiền cho đơn đã nhận.
- Tại sao tương đồng: Cả hai cùng nói về thao tác yêu cầu trả hàng hoặc hoàn tiền cho đơn mua.

**Ví dụ có độ tương tự THẤP:**

- Câu A: Shipper sẽ liên hệ người mua ba lần để giao hàng.
- Câu B: Python dùng thụt lề để xác định khối lệnh.
- Tại sao khác: Hai câu thuộc hai chủ đề không liên quan: giao nhận thương mại điện tử và lập trình.

**Tại sao độ tương tự cosine (cosine similarity) được ưu tiên hơn khoảng cách Euclid (Euclidean distance) cho text embeddings?**

Cosine so sánh hướng của vector nên ít bị ảnh hưởng bởi độ lớn vector, vốn có thể thay đổi theo độ dài câu hoặc cách mô hình chuẩn hóa. Với embedding văn bản, hướng thường mang thông tin ngữ nghĩa hữu ích hơn độ lớn tuyệt đối.

### Bài toán tính toán Chunking (Bài tập 1.2)

**Tài liệu 10,000 ký tự, `chunk_size=500`, `overlap=50`. Bao nhiêu chunks?**

Với bước dịch `step = 500 - 50 = 450`, chunk đầu tiên bắt đầu tại vị trí `0`. Số chunk là `ceil((10,000 - 500) / 450) + 1 = ceil(21.11) + 1 = 23`.

**Đáp án:** 23 chunks.

**Nếu độ chồng chéo (overlap) tăng lên 100, số lượng chunk thay đổi thế nào? Tại sao muốn độ chồng chéo nhiều hơn?**

Khi đó `step = 400`, nên số chunk là `ceil(9,500 / 400) + 1 = 25`; số chunk tăng từ 23 lên 25. Overlap lớn hơn giữ lại nhiều ngữ cảnh ở ranh giới chunk, giảm nguy cơ một ý hoặc câu bị tách mất thông tin, đổi lại là tốn lưu trữ và truy xuất hơn.

---

## 2. Hướng tiếp cận của tôi (My Approach) — Cá nhân (10 điểm)

### Các hàm chia nhỏ (Chunking Functions)

**`SentenceChunker.chunk` — hướng tiếp cận:**

Tôi dùng regex `(?<=[.!?])\s+` để cắt tại khoảng trắng theo sau dấu chấm, chấm than hoặc chấm hỏi; nhờ lookbehind, dấu kết câu vẫn thuộc về câu phía trước. Hàm loại chuỗi rỗng và `strip()` từng câu/chunk để xử lý văn bản chỉ có khoảng trắng, nhiều dòng trống hoặc khoảng trắng dư. Sau đó các câu được ghép theo số câu tối đa đã cấu hình.

**`RecursiveChunker.chunk` / `_split` — hướng tiếp cận:**

Thuật toán ưu tiên lần lượt `\n\n`, `\n`, `. `, khoảng trắng rồi đến cắt ký tự; nếu đoạn vẫn quá dài sau khi tách ở mức hiện tại thì đệ quy với các separator có ưu tiên thấp hơn. Trường hợp cơ sở là đoạn rỗng, đoạn đã không dài quá `chunk_size`, hoặc không còn separator/đến separator rỗng: khi đó hàm trả về rỗng, trả nguyên đoạn, hoặc cắt cứng theo `chunk_size`. Các mảnh nhỏ được ghép lại khi vẫn không vượt giới hạn.

### Lớp EmbeddingStore

**`add_documents` + `search` — hướng tiếp cận:**

Mỗi `Document` được chuyển thành record gồm `id`, `content`, bản sao metadata và vector embedding; record được thêm vào ChromaDB nếu thư viện khả dụng, nếu không thì lưu trong danh sách bộ nhớ. Khi tìm kiếm, query cũng được embed và so với các embedding đã lưu; nhánh bộ nhớ xếp hạng bằng dot product (vì `MockEmbedder` đã chuẩn hóa vector), còn ChromaDB trả distance và mã chuyển thành score `-distance`.

**`search_with_filter` + `delete_document` — hướng tiếp cận:**

Việc lọc diễn ra **trước** khi tính/xếp hạng similarity: nhánh bộ nhớ chỉ giữ các record có mọi cặp metadata khớp `metadata_filter`, còn ChromaDB nhận điều kiện `where`. Khi xóa, hàm xóa toàn bộ chunk có `metadata.doc_id` đúng bằng `doc_id`; với bộ nhớ, danh sách được thay bằng các record còn lại và hàm trả về việc kích thước có thay đổi hay không.

### Tác tử KnowledgeBaseAgent

**`answer` — hướng tiếp cận:**

Agent truy xuất `top_k` chunk trước, đánh số chúng từ `[1]` rồi nối bằng dòng trống để tạo context. Prompt yêu cầu LLM chỉ sử dụng context, nói rõ khi context không đủ và trích nguồn theo số thứ tự khi có thể; cuối prompt đặt câu hỏi người dùng và nhãn `Trả lời:`. Nếu không có chunk nào, agent trả ngay thông báo không tìm thấy thông tin phù hợp.

---

## 3. Hoàn thiện code (Core Implementation) — Cá nhân (30 điểm)

Vượt qua bộ kiểm thử là điều kiện tính điểm phần này.

### Kết Quả Kiểm Thử (Test Results)

```text
$ .venv/bin/python -m pytest tests/ -v
============================= test session starts ==============================
platform linux -- Python 3.12.3, pytest-9.1.1
collected 44 items

tests/test_bench.py ..
tests/test_solution.py ..........................................

============================== 44 passed in 0.46s ==============================
```

**Số lượng bài test vượt qua (pass):** 44 / 44

> Khung đề bài nêu 42 test, nhưng phiên bản repository hiện tại có thêm 2 test hồi quy cho `bench.py`; vì vậy kết quả được ghi theo bộ test thực tế đã chạy.

---

## 4. Dự đoán độ tương tự (Similarity Predictions) — Cá nhân (5 điểm)

Các điểm thực tế dưới đây được tính bằng `MockEmbedder` và `compute_similarity` ngày 20/09/2026. `MockEmbedder` sinh vector xác định từ MD5 để phục vụ kiểm thử, nên bảng này phản ánh đúng phép tính cosine nhưng không phải phép đo ngữ nghĩa đáng tin cậy của mô hình embedding thật.

| Cặp | Câu A | Câu B | Dự đoán | Điểm thực tế | Đúng? |
| --- | ----- | ----- | -------- | ------------ | ----- |
| 1 | Đơn đã hủy không thể khôi phục. | Đơn `Đã hủy` không được giao lại. | cao | -0.0416 (thấp) | Không |
| 2 | Yêu cầu được xử lý trong 3–5 ngày. | Hoàn tiền trong 1–14 ngày. | thấp | -0.2707 (thấp) | Có |
| 3 | Shipper liên hệ 3 lần. | Chờ phản hồi Người bán khi `Chờ lấy hàng`. | thấp | 0.1274 (cao hơn các cặp khác) | Không |
| 4 | Chọn lý do `Chưa nhận được hàng`. | Không cần bằng chứng cho lý do này. | cao | -0.0399 (thấp) | Không |
| 5 | `Chờ xác nhận` hủy ngay. | Đơn chuyển hoàn không thể giao lại. | thấp | -0.0951 (thấp) | Có |

**Kết quả nào bất ngờ nhất? Điều này nói gì về cách embeddings biểu diễn ý nghĩa?**

Điểm cao nhất lại là cặp 3, dù hai câu thuộc hai quy trình khác nhau, trong khi các cặp 1 và 4 có quan hệ ngữ nghĩa gần lại cho điểm âm. Điều này không phản ánh đặc tính chung của semantic embedding; nó cho thấy vector giả lập dựa trên MD5 không mã hóa ý nghĩa câu, nên chỉ phù hợp kiểm thử tính đúng đắn của pipeline và công thức cosine. Vì vậy benchmark truy xuất của tôi dùng `gemini-embedding-001` thay vì `MockEmbedder`.

---

## 5. Kết quả truy xuất của tôi (Competition Results) — Cá nhân (10 điểm)

Tôi chạy đúng 5 câu hỏi đánh giá chung trong `REPORT_NHOM.md` trên corpus `data/shopee-buyer-policy/`, với `RecursiveChunker(chunk_size=700)`, Gemini `gemini-embedding-001` và `top_k=3`. Corpus tạo 27 chunks.

| # | Câu hỏi (Query) | Top-1 Chunk truy xuất được (tóm tắt) | Điểm Score | Có liên quan không? (Relevant) | Câu trả lời của Agent (tóm tắt) |
| - | --------------- | ------------------------------------ | ---------- | ------------------------------ | ------------------------------- |
| 1 | Người mua gửi yêu cầu Trả hàng/Hoàn tiền trực tiếp tại trang đơn hàng như thế nào? | `shopee-buyer-return-refund-request`: hướng dẫn gửi yêu cầu Trả hàng/Hoàn tiền. | 0.8221 | Có (top-3 có evidence; top-1 mới là phần giới thiệu) | Nêu đây là hướng dẫn yêu cầu trả hàng/hoàn tiền, nhưng chưa nêu đủ đường dẫn thao tác. |
| 2 | Yêu cầu Trả hàng/Hoàn tiền thường được xử lý trong bao lâu và tiền hoàn được nhận sau bao lâu nếu yêu cầu được chấp nhận? | `shopee-buyer-return-refund-request`: thời gian hoàn tiền 1–14 ngày làm việc. | 0.8010 | Có | Nêu đúng 1–14 ngày tùy phương thức thanh toán, nhưng thiếu mốc xử lý 3–5 ngày. |
| 3 | Người mua có thể yêu cầu hủy đơn ở những trạng thái nào? | `shopee-buyer-order-cancellation`: các trạng thái hủy đơn. | 0.8339 | Có (top-3 có evidence) | Chỉ trích heading/mở đầu nên chưa nêu đầy đủ trạng thái và điều kiện. |
| 4 | Nếu Shipper không liên hệ nhưng cập nhật giao hàng không thành công, Người mua cần làm gì? | `shopee-buyer-delivery-status-issue`: trường hợp giao không thành công. | 0.8283 | Có | Đúng: chờ cuộc gọi tiếp theo; Shipper sẽ liên hệ tối đa 3 lần. |
| 5 | Nếu quá 24 giờ đơn bị cập nhật đã giao nhưng Người mua chưa nhận được hàng, cần chọn lý do Trả hàng/Hoàn tiền nào và có cần bằng chứng không? | `shopee-buyer-delivery-status-issue`: đơn cập nhật đã giao nhưng chưa nhận hàng. | 0.7851 | Có (top-3 có evidence) | Đúng: chọn `Chưa nhận được hàng` và không cần cung cấp bằng chứng. |

**Bao nhiêu câu hỏi trả về chunk có liên quan trong top-3?** 5 / 5

**Điều hay nhất tôi học được từ thành viên khác / nhóm khác (qua demo):**

Qua đối chiếu các cấu hình trong nhóm, tôi thấy top-3 đã có evidence không đồng nghĩa agent sẽ trả lời đầy đủ: với Recursive, Q1, Q2 và Q3 vẫn thiếu chi tiết vì hàm extractive ưu tiên phần đầu chunk. So sánh Fixed (6/10), Recursive (7/10) và Sentence (7/10) cũng cho thấy cần đánh giá cả evidence, câu trả lời cuối và số chunk thay vì chỉ chọn chiến lược có score top-1 cao nhất.

---

## Tự Đánh Giá (Phần Cá Nhân)

| Tiêu chí | Điểm tự đánh giá |
| --- | ---: |
| Khởi động (Warm-up) | 5 / 5 |
| Hướng tiếp cận của tôi (My Approach) | 10 / 10 |
| Hoàn thiện code (Core Implementation — tests) | 30 / 30 |
| Dự đoán độ tương tự (Similarity Predictions) | 5 / 5 |
| Kết quả truy xuất của tôi (Competition Results) | 10 / 10 |
| **Tổng phần cá nhân** | **60 / 60** |
