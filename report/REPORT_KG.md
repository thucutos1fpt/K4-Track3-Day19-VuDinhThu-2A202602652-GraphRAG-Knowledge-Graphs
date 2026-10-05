# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Vũ Đình Thư  **MSSV:** 2A202602652  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     46.8
graph       196     91958     4583   0.00925    127.3

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       48   0.00013     1.32
graph       0.69   1.50     3194       79   0.00052     2.01
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00925 | ×8.26 |
| Indexing giây | 46.8 | 127.3 | ×2.72 |
| Mỗi câu: USD | $0.00013 | $0.00052 | ×4.00 |
| Mỗi câu: giây | 1.32 | 2.01 | ×1.52 |
| Mỗi câu: in_tok | 694 | 3194 | ×4.60 |

**Chi phí tăng thêm đến từ đâu?** (2–3 câu)
> GraphRAG tốn thêm chi phí indexing vì gọi LLM trích xuất 20 bài tin tức và ghi graph, trong khi Flat RAG chỉ tạo embedding. Ở lúc hỏi, graph facts làm prompt dài hơn 4.59 lần; đổi lại chúng đưa Điều luật và khoản luật từ KB còn lại vào ngữ cảnh.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Cả hai đều truy hồi trực tiếp được định nghĩa tiền chất trong luật. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Tên hai bị cáo nằm ngay trong bài báo được truy hồi. |
| Q3 | cross-kb | 0.00 / 0 | 1.00 / 2 | Graph | Graph nối Lê Minh Thành sang Điều 251 và khoản 1. |
| Q4 | cross-kb | 0.00 / 0 | 0.33 / 1 | Graph (một phần) | Graph lấy được hành vi, nhưng không lấy khung cao nhất. |
| Q5 | cross-kb-multi-hop | 0.60 / 1 | 0.80 / 1 | Graph (recall) | Graph có khoản 4 và khung án, nhưng gắn sai Điều 251 thay vì Điều 250. |
| Q6 | aggregation | 0.00 / 1 | 0.00 / 1 | Hòa | Cả hai chưa nêu đúng bộ tên vụ việc theo đáp án chuẩn. |

## 3. Phân tích lỗi (20 điểm)

Chọn ít nhất 2 nhóm lỗi trong E1–E6 (`LAB_GUIDE.md` Bước 8.4). Sao chép khung dưới đây cho mỗi lỗi.

### Lỗi E1: Liên kết tội danh sai dẫn đến sai Điều luật (Q5)

- **Hiện tượng:** GraphRAG trả lời được khoản 4 và khung hình phạt cho Cái Quang Huy, nhưng ghi nhầm Điều 251; đáp án chuẩn cần Điều 250.
- **Bằng chứng:** Trong `ket_qua_benchmark_kg.txt`, Q5 GraphRAG có `recall=0.80 judge=1` và trả lời: “khoản áp dụng tương ứng là khoản 4 của Điều 251 BLHS”.

```cypher
MATCH (k:Case)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(a:Article)
WHERE k.name CONTAINS 'Cái Quang Huy'
RETURN k.name, c.name, a.id;
```

```
Kết quả cho thấy Case được nối qua `Crime` tới Điều 251, trong khi tội “vận chuyển trái phép chất ma túy” phải nối tới Điều 250.
```

- **Nguyên nhân:** Lỗi ở bước LLM extraction/entity linking: tội danh của bài báo không được chuẩn hóa chính xác về tên chuẩn trong luật trước khi MERGE node `Crime`.
- **Đề xuất sửa:** Tăng độ ràng buộc trong prompt, lưu thêm evidence span từ bài báo, và kiểm tra chéo tên `Crime` với điều số trong nguồn pháp luật trước khi ghi graph.

### Lỗi E2: Câu hỏi tổng hợp bị giới hạn bởi vector top-k (Q6)

- **Hiện tượng:** GraphRAG liệt kê một số vụ có MDMA nhưng không nêu đúng tên thực thể bắt buộc trong đáp án, nên Q6 vẫn có `recall=0.00`.
- **Bằng chứng:** Kết quả Q6 GraphRAG có `recall=0.00 judge=1`; câu trả lời dùng các tên tóm tắt như “Vụ vận chuyển ma túy từ Đức về Việt Nam” thay vì các tên chuẩn yêu cầu như “Cái Quang Huy”, “Lê Minh Thành”, “Pháp y tâm thần”.

```cypher
MATCH (k:Case)-[:INVOLVES]->(s:Substance {name:'MDMA'})
RETURN k.name, k.source_title;
```

```
Truy vấn graph trả về nhiều Case MDMA, trong đó có các vụ liên quan Viện Pháp y tâm thần; nhưng thuộc tính `Case.name` là tóm tắt do LLM sinh nên không đảm bảo giữ tên người/vụ cần cho phép đo keyword recall.
```

- **Nguyên nhân:** Ngoài giới hạn `top_k=3`, các Case được đặt tên tự do bởi LLM. Câu trả lời có thể đúng ngữ nghĩa nhưng không chứa từ khóa tên riêng dùng cho benchmark.
- **Đề xuất sửa:** Phát hiện câu hỏi aggregation và quét trực tiếp các Case theo `Substance`; đồng thời lưu người/tên riêng trong facts trả về thay vì chỉ dùng `Case.name` do LLM đặt.

## 4. Kết luận (5 điểm)

Khi nào nên dùng KG, khi nào Flat RAG là đủ? Dẫn số liệu ở mục 1–2.
> Nên dùng Knowledge Graph khi câu hỏi cần nối nhiều nguồn hoặc nhiều hop, nhất là quan hệ vụ án → tội danh → Điều luật → khoản. Trong benchmark này, GraphRAG tăng recall trung bình từ 0.43 lên 0.69 và judge từ 1.00 lên 1.50; riêng Q3 tăng từ 0 lên 1.00. Flat RAG đủ cho câu một nguồn như Q1 và Q2, vì kết quả hai pipeline đều đạt recall 1.00/judge 2, trong khi Flat rẻ hơn: indexing thấp hơn 8.26 lần và chi phí mỗi câu thấp hơn 4.00 lần.

## 5. Tự kiểm (5 điểm)

```
$ python -m pytest tests/ -q
................................................                         [100%]
48 passed in 0.20s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00065
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Dương Minh Tuấn (Hoàng Nato) hoặc Lê Minh Thành.

## Vấn đề gặp phải (không tính điểm)

Lỗi chưa giải quyết được: lệnh đã chạy, toàn bộ thông báo lỗi, những gì đã thử.
> Lỗi Unicode khi chạy benchmark trên Windows được xử lý bằng `$env:PYTHONUTF8='1'` trước khi chạy Python.
