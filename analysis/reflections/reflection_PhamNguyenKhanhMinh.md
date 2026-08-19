# Reflection — Lab 18: Production RAG

**Họ tên:** Phạm Nguyễn Khánh Minh

---

### Phần 1: Mapping bài giảng

| Lecture Concept | Module | Hàm cụ thể | Observation |
|----------------|--------|-------------|-------------|
| Semantic chunking | M1 | `chunk_semantic()` | Threshold 0.85 giúp chia các câu cùng chủ đề, giảm tình trạng đứt mạch so với basic chunking chia theo đoạn. |
| Hierarchical chunking | M1 | `chunk_hierarchical()` | Kỹ thuật Parent-Child giúp tìm kiếm chính xác (trên child) nhưng LLM vẫn có ngữ cảnh rộng (từ parent). |
| BM25 + Dense fusion | M2 | `reciprocal_rank_fusion()` | RRF giải quyết tốt việc BM25 giỏi tìm keyword (như tên riêng, mã số) còn Dense giỏi tìm ngữ nghĩa. |
| Cross-encoder reranking | M3 | `CrossEncoderReranker.rerank()` | Giúp đẩy context chính xác nhất lên top 3, loại bỏ các chunk bị nhiễu do Dense model kéo về. |
| RAGAS 4 metrics | M4 | `evaluate_ragas()` | Giúp tự động chấm điểm Pipeline mà không cần manual grading. Dễ nhận diện lỗi từ Generator (Faithfulness thấp) hay Retriever (Context Precision thấp). |
| Contextual embeddings | M5 | `contextual_prepend()` | Prepend title/context vào đầu chunk giúp model hiểu chunk đang nói về văn bản/chính sách nào (giảm 49% lỗi mất bối cảnh). |

### Phần 2: Khó khăn & giải quyết

- **Khó khăn 1:** Khi chạy BM25 với tiếng Việt, nếu không segment từ (tokenization) bằng `underthesea`, kết quả bị sai lệch vì BM25 tách theo khoảng trắng (ví dụ "nhân viên" thành "nhân" và "viên").
  - **Cách debug & giải quyết:** Sử dụng hàm `segment_vietnamese` để chuyển "nhân viên" thành "nhân_viên" trước khi nạp vào mảng corpus của BM25.
- **Khó khăn 2:** Gọi OpenAI API trong `m5_enrichment.py` cho từng chunk bị chậm và dễ hit rate limit.
  - **Cách giải quyết:** Implement hàm `_enrich_single_call()` kết hợp summary, HyQA, metadata extraction vào chung 1 prompt để giảm số lượng API calls (tối ưu cost/latency).

### Phần 3: Action Plan cho project

## Project: Chatbot nội bộ tư vấn chính sách Nhân Sự & IT

### Hiện tại
- RAG pipeline hiện tại: Chỉ dùng Naive RAG (Basic recursive character chunking + OpenAI text-embedding-3-small).
- Known issues: Thường xuyên trả lời sai khi hỏi về các policy có nhiều phiên bản (v1, v2); không trả lời được câu hỏi cần nhiều ngữ cảnh.

### Plan áp dụng
1. [x] **Chunking strategy:** Áp dụng Hierarchical Chunking (Parent 2048 - Child 256) để tăng độ chính xác khi query nhưng giữ bối cảnh đầy đủ cho LLM đọc.
2. [x] **Search:** Dùng Hybrid Search (BM25 + Dense) để vừa bắt keyword (như "PVI", "MFA"), vừa hiểu ngữ nghĩa.
3. [x] **Reranking:** Có sử dụng model Cross-encoder `bge-reranker-v2-m3` để lọc top 3 trả về cho LLM.
4. [x] **Evaluation:** Dùng RAGAS chạy CI/CD mỗi khi có data mới vào hệ thống.
5. [x] **Enrichment:** Sử dụng Contextual prepend để luôn gắn tên tài liệu (ví dụ "Sổ tay nhân viên 2024") vào mỗi chunk.

### Timeline
- **Tuần 1:** Triển khai lại hàm Chunking (M1) và Embedding sang Qdrant.
- **Tuần 2:** Setup BM25 + Qdrant (M2) và ghép Reranker (M3).
- **Tuần 3:** Tích hợp RAGAS Eval (M4) và tinh chỉnh prompt dựa trên failure analysis.
