# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Đặng Hữu Cương  **MSSV:** 2A202602572  **Ngày:** 05/10/2026

> Kỳ vọng và thang điểm: `SUBMISSION.md`. Mọi số liệu phải khớp với `ket_qua_benchmark_kg.txt`. Bản thiết kế ontology nộp riêng ở `report/ONTOLOGY.md`.

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```text
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176         0        0   0.00000    112.2
graph       196     34619     5707   0.00574    177.5

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.51   1.50      696       72   0.00010     4.54
graph       1.00   2.00     5236      138   0.00058     3.07
```

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00000 | $0.00574 | ∞ (hoặc +$0.00574) |
| Indexing giây | 112.2s | 177.5s | ×1.58 |
| Mỗi câu: USD | $0.00010 | $0.00058 | ×5.80 |
| Mỗi câu: giây | 4.54s | 3.07s | ×0.68 (Graph nhanh hơn) |
| Mỗi câu: in_tok | 696 | 5,236 | ×7.52 |

**Chi phí tăng thêm đến từ đâu?**
> Chi phí tăng thêm ở pha **Indexing** chủ yếu đến từ 20 lần gọi LLM (`gemini-3.5-flash-lite`) để trích xuất cấu trúc quan hệ ngữ nghĩa (thực thể, vai trò, tội danh, mức án) từ 20 bài báo tin tức dưới dạng JSON, tiêu tốn 34.619 input tokens và 5.707 output tokens. Ở pha **Querying**, chi phí mỗi câu hỏi của GraphRAG cao hơn khoảng 5.8 lần do prompt ngữ cảnh được bổ sung các facts mở rộng đa chặng (multi-hop) từ đồ thị Neo4j (dẫn chứng điều luật, các khoản quy định khung hình phạt), làm số lượng input token trung bình tăng từ 696 lên 5.236 tokens. Đổi lại, độ trễ sinh câu trả lời của GraphRAG giảm từ 4.54s xuống 3.07s do ngữ cảnh đồ thị có tính chọn lọc cao, giúp LLM tổng hợp đáp án dứt khoát hơn.

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại | Flat recall / judge | Graph recall / judge | Thắng | Vì sao (1 câu) |
| --- | --- | --- | --- | --- | --- |
| Q1 | single-hop-law | 1.00 / 2 | 1.00 / 2 | Hòa | Định nghĩa tiền chất nằm gọn trong một đoạn của Luật PCMT nên vector search thuần túy đã lấy đủ thông tin. |
| Q2 | single-hop-news | 1.00 / 2 | 1.00 / 2 | Hòa | Thông tin mức án tử hình nằm tập trung trong 1 bài báo xét xử 36kg ma túy nên Flat RAG tìm kiếm chính xác. |
| Q3 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Mức án của Lê Minh Thành nằm ở tin tức còn khung hình phạt khoản 1 nằm ở Điều 251 BLHS; Flat RAG không thể nối 2 nguồn văn bản độc lập này. |
| Q4 | cross-kb | 0.33 / 1 | 1.00 / 2 | Graph | Flat RAG chỉ tìm thấy hành vi của "Hoàng Nato" nhưng thiếu điều luật, trong khi GraphRAG đi qua node Crime để trích xuất mức án tối đa tại khoản 4 Điều 255. |
| Q5 | cross-kb-multi-hop | 0.40 / 1 | 1.00 / 2 | Graph | Vụ Cái Quang Huy yêu cầu đối chiếu khối lượng >9,6kg MDMA với khoản 4 Điều 250; GraphRAG truy xuất chính xác khoản luật chứa chất MDMA tương ứng. |
| Q6 | aggregation | 0.00 / 2 | 1.00 / 2 | Graph | Câu hỏi tổng hợp đòi hỏi quét qua nhiều vụ án liên quan đến MDMA; Flat RAG chỉ lấy được 3 đoạn nhỏ rời rạc không đủ từ khóa bắt buộc, trong khi GraphRAG gom đủ các vụ từ quan hệ `INVOLVES`. |

## 3. Phân tích lỗi (20 điểm)

### Lỗi E1: Cầu nối gãy (Broken Bridge giữa Tin tức và Luật)

- **Hiện tượng:** Một số vụ án trong KB tin tức không có quan hệ `CHARGED_WITH` kết nối sang node `Crime` của KB luật, dẫn tới việc không thể mở rộng truy vấn sang điều luật quy định tương ứng.
- **Bằng chứng:** Truy vấn Cypher tìm các vụ án không nối sang node tội danh:

```cypher
MATCH (k:Case) WHERE NOT (k)-[:CHARGED_WITH]->() RETURN k.name, k.doc_id;
```

```text
- Vụ vận chuyển hơn 800kg chất nghi ma túy tại Preah Sihanouk (doc_id: news-100260924145818945)
- Vụ tông cảnh sát giao thông tại An Giang (doc_id: news-100260926112415229)
- Triệt phá chuyên án A3-626P (doc_id: news-100261002184934505)
```

- **Nguyên nhân:** Nằm ở bản chất văn bản tin tức: Các bài báo này phản ánh sự việc đang trong giai đoạn nóng (đang truy đuổi, bắt quả tang vận chuyển "chất nghi là ma túy") hoặc vụ việc xảy ra ở nước ngoài (Campuchia), chưa có kết luận giám định chính thức hay quyết định khởi tố theo tội danh chuẩn trong BLHS Việt Nam. Do đó, danh sách tội danh trong prompt không khớp và `link_entity` trả về `None`.
- **Đề xuất sửa:** Bổ sung cơ chế bắc cầu dự phòng (fallback bridge) thông qua node chất ma túy `Substance`: nếu `Case` chưa có `Crime`, Cypher vẫn đi tiếp từ `(Case)-[:INVOLVES]->(Substance)<-[:MENTIONS]-(Clause)<-[:HAS_CLAUSE]-(Article)` để cung cấp ngữ cảnh các điều luật có điều chỉnh chất ma túy đó. Đánh đổi: số lượng facts trả về tăng lên, làm tăng chi phí token đầu vào.

---

### Lỗi E3: Trùng thực thể (Entity Duplication trên node Case)

- **Hiện tượng:** Cùng một vụ án ngoài đời thực bị phân mảnh thành nhiều node `Case` riêng biệt trên đồ thị Neo4j.
- **Bằng chứng:** Truy vấn Cypher danh sách các vụ án liên quan đến đối tượng "Hoàng Nato":

```cypher
MATCH (k:Case) WHERE k.name CONTAINS "Hoàng Nato" RETURN k.name, k.doc_id;
```

```text
- Vụ triệt phá 8 đường dây ma túy liên quan 'Hoàng Nato' tại TP.HCM (doc_id: news-100260920221957595)
- Vụ triệt phá 8 đường dây ma túy liên quan đến 'Hoàng Nato' tại TP.HCM (doc_id: news-100260925144412498)
- Vụ triệt phá 8 đường dây ma túy liên quan đến TikToker Phannhibeauty và Hoàng Nato tại TP.HCM (doc_id: news-100260922111804786)
```

- **Nguyên nhân:** Do thiết kế ontology dùng thuộc tính `name` làm khóa định danh (`MERGE (k:Case {name: $name})`). Do mỗi bài báo có tiêu đề và văn phong khác nhau, LLM khi trích xuất đã tự sinh các chuỗi tên vụ án có sự sai lệch nhỏ (thêm chữ "đến", bổ sung thêm tên "TikToker Phannhibeauty"), khiến câu lệnh `MERGE` coi đây là 3 vụ án khác nhau dù cùng phản ánh chuyên án triệt phá 8 đường dây ma túy tại TP.HCM.
- **Đề xuất sửa:** 
  1. Chuẩn hóa tên vụ án thông qua một hàm entity resolution tương tự `link_entity` trước khi `MERGE` vào Neo4j.
  2. Bổ sung liên kết tương đương `SAME_AS` giữa các vụ án nếu có cùng danh sách bị can chính (`Person`) và cùng địa bàn (`Location`). Đánh đổi: Cần thêm một bước phân giải thực thể sau khi trích xuất, làm tăng thêm thời gian indexing.

---

### Lỗi E6: Thuộc tính thiếu trên quan hệ INVOLVED_IN

- **Hiện tượng:** Nhiều quan hệ `INVOLVED_IN` giữa `Person` và `Case` có thuộc tính `charge` bị rỗng (`charge = ''`).
- **Bằng chứng:** Truy vấn Cypher các cá nhân có `charge` rỗng:

```cypher
MATCH (p:Person)-[r:INVOLVED_IN]->(k:Case) WHERE r.charge = '' 
RETURN p.name, r.role, k.name LIMIT 5;
```

```text
- Nguyễn Hữu Đức | role: 'người liên quan' | case: Vụ vận chuyển hơn 10kg ma túy từ Đức về Việt Nam...
- Ngô Văn Vinh | role: 'cán bộ' | case: Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương...
- Trần Văn Trường | role: 'cán bộ' | case: Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương...
- Trần Quốc An | role: 'cán bộ' | case: Vụ án sai phạm tại Viện Pháp y tâm thần Trung ương...
```

- **Nguyên nhân:**
  - *Trường hợp hợp lý:* Các đối tượng có vai trò là `'cán bộ'` (giám định viên, điều tra viên) hoặc `'người liên quan'` không phải là người thực hiện hành vi phạm tội nên việc không có tội danh cáo buộc (`charge = ''`) là hoàn toàn phản ánh đúng sự thật khách quan của tố tụng hình sự.
  - *Trường hợp thiếu sót:* Một số đối tượng là `'bị can'` trong bài báo tổng hợp (nhiều tội danh trong cùng vụ án) bị LLM bỏ sót tội danh cá nhân do bài viết không gán nhãn chi tiết từng người.
- **Đề xuất sửa:** Trong prompt trích xuất, phân tách rõ ràng hướng dẫn: nếu là `cán bộ`/`người liên quan` thì để `charge: null`, còn nếu là `bị can`/`bị cáo` thì bắt buộc phải suy luận tội danh chính của vụ án nếu không có tội danh riêng biệt.

## 4. Kết luận (5 điểm)

Từ số liệu đo đạc thực nghiệm ở mục 1 và 2:
1. **Khi nào Flat RAG là đủ:** Đối với các tác vụ truy vấn đơn chặng (single-hop) như Q1 (định nghĩa trong luật) và Q2 (danh sách tử hình trong 1 phiên tòa), Flat RAG đạt độ chính xác tối đa (`recall = 1.00`, `judge = 2.00`) với chi phí cực thấp ($0.00010/câu) và không mất chi phí xây dựng Knowledge Graph ban đầu. Khi dữ liệu có tính cô đọng, độc lập và câu trả lời nằm trọn trong một đoạn văn, Flat RAG là phương án tối ưu về mặt kinh tế và triển khai.
2. **Khi nào nên dùng Knowledge Graph (GraphRAG):**
   - **Bắt buộc dùng khi câu hỏi đòi hỏi liên kết thông tin xuyên cơ sở tri thức (Cross-KB):** Ở các câu Q3, Q4, Q5 (nối giữa lời khai/bản án trong báo chí với điều luật và khung hình phạt trong BLHS), Flat RAG hoàn toàn thất bại (`recall` chỉ đạt 0.33 – 0.40, `judge = 1.00`), trong khi GraphRAG đạt độ chính xác tuyệt đối 100% (`recall = 1.00`, `judge = 2.00`).
   - **Vượt trội ở câu hỏi tổng hợp (Aggregation):** Ở câu Q6, Flat RAG đạt `recall = 0.00` do giới hạn `top_k=3` không thể bao quát toàn bộ tài liệu rải rác, trong khi GraphRAG liên kết đồ thị thu thập đầy đủ 100% các vụ án liên quan đến chất MDMA.
   - **Đánh đổi chi phí:** Chi phí indexing ban đầu cho KG chỉ tốn ~$0.00574 (chưa tới 150 VNĐ) và chi phí mỗi câu tăng thêm ~$0.00048, nhưng đổi lại chất lượng câu trả lời nhảy vọt từ mức không dùng được (sai điều luật, thiếu khung hình phạt) lên mức hoàn chỉnh, chuẩn xác pháp lý.

## 5. Tự kiểm (5 điểm)

```text
$ pytest tests/ -q
................................................                         [100%]
48 passed in 1.29s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = gemini:gemini-3.5-flash-lite | embedding = gemini:gemini-embedding-001
[OK] KG-2 build_graph: 148 node / 294 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 23 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00055. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

Ảnh Neo4j: `report/img/kg_count.png`, `report/img/kg_cross_kb.png`, `report/img/kg_my_case.png`.
Người đã chọn cho `kg_my_case.png`: Cái Quang Huy (bị cáo trong vụ án vận chuyển hơn 9,6kg ma túy MDMA qua sân bay Nội Bài).

## Vấn đề gặp phải (không tính điểm)

1. Ban đầu OpenAI API hết credit (`RateLimitError 429 credit_balance_exhausted`), sau đó chuyển sang Google Gemini API.
2. Model cũ `gemini-2.5-flash-lite` và `gemini-2.0-flash` trên Google AI Studio đã bị deprecate (`NotFoundError 404`). Đã chuyển sang `gemini-3.5-flash-lite` và `gemini-embedding-001`, bổ sung cơ chế retry cho các lỗi quá tải tạm thời (503 / 429), giúp toàn bộ hệ thống hoạt động ổn định và vượt qua mọi bài test.
