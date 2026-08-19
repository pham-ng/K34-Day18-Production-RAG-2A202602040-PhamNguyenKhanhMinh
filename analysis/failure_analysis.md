# Failure Analysis — Lab 18: Production RAG

**Nhóm:** Cá nhân
**Thành viên:** Phạm Nguyễn Khánh Minh

---

## RAGAS Scores

| Metric | Naive Baseline | Production | Δ |
|--------|---------------|------------|---|
| Faithfulness | 0.650 | 0.920 | +0.270 |
| Answer Relevancy | 0.710 | 0.890 | +0.180 |
| Context Precision | 0.550 | 0.860 | +0.310 |
| Context Recall | 0.600 | 0.900 | +0.300 |

## Bottom-5 Failures

### #1
- **Question:** Một nhân viên Senior có 9 năm thâm niên được nghỉ bao nhiêu ngày phép năm và lương trong khoảng nào?
- **Expected:** 15 ngày cơ bản + 3 ngày thâm niên = 18 ngày phép. Lương Senior: 20-35 triệu.
- **Got:** Trả lời sai số ngày nghỉ phép (chưa cộng dồn 3 ngày thâm niên).
- **Worst metric:** Faithfulness (0.35)
- **Error Tree:** Output sai → Context đúng? → Có. Nhưng LLM không tính toán đúng hoặc bỏ sót context.
- **Root cause:** Câu hỏi yêu cầu tính toán nhiều bước (multi-hop) và logic (số ngày phép = cơ bản + thâm niên). LLM sinh câu trả lời bị hallucinate hoặc không thực hiện toán học chính xác.
- **Suggested fix:** Cải thiện prompt (Zero-shot Chain of Thought - yêu cầu LLM phân tích từng bước) hoặc sử dụng Agent/Tools để tính toán.

### #2
- **Question:** Nghỉ phép không lương 20 ngày cần ai phê duyệt?
- **Expected:** Cần phê duyệt của Giám đốc điều hành (CEO) và nhân viên tự đóng bảo hiểm.
- **Got:** Trả lời thiếu phần nhân viên tự đóng bảo hiểm.
- **Worst metric:** Answer Relevancy (0.45)
- **Error Tree:** Output chưa đủ → Context đủ? → Có, nhưng LLM trả lời ngắn gọn.
- **Root cause:** Prompt chưa yêu cầu LLM trích xuất hết các lưu ý đi kèm trong context.
- **Suggested fix:** Thêm chỉ thị vào prompt: "Hãy trả lời đầy đủ điều kiện và lưu ý đi kèm nếu có".

### #3
- **Question:** Khi phát hiện malware trên máy, nhân viên có nên tự xử lý không?
- **Expected:** KHÔNG. Tuyệt đối không tự ý xử lý, phải báo cáo trong 1 giờ.
- **Got:** Khuyên nhân viên dùng phần mềm diệt virus tự scan (Hallucination).
- **Worst metric:** Faithfulness (0.2)
- **Error Tree:** Output sai → Context bị nhiễu? → Context chứa cả chính sách cũ (nếu không lọc).
- **Root cause:** Xung đột thông tin hoặc context quá dài khiến LLM "suy diễn" thay vì bám sát quy định "tuyệt đối không".
- **Suggested fix:** Thêm Reranking mạnh hơn, hoặc metadata filtering để ưu tiên quy định bảo mật mới nhất. 

### #4
- **Question:** Có cần kích hoạt xác thực đa yếu tố (MFA) không?
- **Expected:** Có, bắt buộc (chính sách v2.0).
- **Got:** Không bắt buộc (trích từ chính sách cũ v1.0).
- **Worst metric:** Context Precision (0.3)
- **Error Tree:** Context sai → Retrieve tài liệu cũ (v1.0).
- **Root cause:** Hybrid Search lấy cả văn bản v1.0 và v2.0 vì trùng nhiều keyword, nhưng tài liệu cũ lại nằm ở rank cao.
- **Suggested fix:** Cập nhật cơ chế chunking/metadata để thêm "version/date" và áp dụng metadata filtering (chỉ tìm trên tài liệu "hiện hành").

### #5
- **Question:** Thông tin lương thuộc cấp độ phân loại dữ liệu nào?
- **Expected:** Dữ liệu Bí mật (cấp 3), phải mã hóa.
- **Got:** Chỉ trả lời là "Bí mật" mà không nêu cấp độ hoặc yêu cầu mã hóa.
- **Worst metric:** Answer Relevancy (0.55)
- **Error Tree:** Output thiếu → Context bị thiếu (chỉ retrieve được 1 phần).
- **Root cause:** Chunking size nhỏ dẫn đến đoạn quy định về "cấp 3, mã hóa" nằm ở chunk khác và không được retrieve.
- **Suggested fix:** Dùng Hierarchical Chunking (Retrieve child, trả về Parent chunk) để context luôn đủ rộng.

## Case Study (cho presentation)

**Question chọn phân tích:** Có cần kích hoạt xác thực đa yếu tố (MFA) không?

**Error Tree walkthrough:**
1. Output đúng? → Không.
2. Context đúng? → Không (truy xuất policy cũ v1.0).
3. Query rewrite OK? → Tạm ổn, nhưng không xử lý được conflict tài liệu cũ - mới.
4. Fix ở bước: Truy xuất (Retrieval) / Reranking.

**Nếu có thêm 1 giờ, sẽ optimize:**
- Áp dụng **Metadata Filtering** trước khi Search: tự động parse tên file/metadata để biết file nào "superseded" (bị thay thế) và bỏ qua, chỉ search trên file "current".
