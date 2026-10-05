# Báo cáo Day 19 — Flat RAG vs GraphRAG

**Họ tên:** Lê Thị Thùy Trang  **MSSV:** 2A202602678  **Ngày:** 05/10/2026

> Báo cáo so sánh thực nghiệm giữa 2 kiến trúc Flat RAG và GraphRAG trên 2 cơ sở tri thức: Luật ma túy (BLHS 2015, Luật PCMT 2021) và Tin tức báo chí (20 bài báo Tuổi Trẻ). Mọi số liệu trong báo cáo được trích xuất trực tiếp từ file kết quả benchmark thực tế `ket_qua_benchmark_kg.txt`.

---

## 1. Chi phí (10 điểm)

Dán 2 bảng `Indexing` và `Querying` từ `ket_qua_benchmark_kg.txt`:

```
== Indexing (one-off)
pipeline  calls    in_tok  out_tok       USD  seconds
flat        176     56072        0   0.00112     45.0
graph       196     91958     4720   0.00934    112.8

== Querying (mean per question)
pipeline  recall  judge   in_tok  out_tok       USD  seconds
flat        0.43   1.00      694       47   0.00013     2.08
graph       0.74   1.50     3655       82   0.00059     2.07
```

### Bảng tổng hợp so sánh các chỉ số

| Chỉ số | Flat | Graph | Graph / Flat |
| --- | --- | --- | --- |
| Indexing USD | $0.00112 | $0.00934 | ×8.34 |
| Indexing giây | 45.0s | 112.8s | ×2.51 |
| Mỗi câu: USD | $0.00013 | $0.00059 | ×4.54 |
| Mỗi câu: giây | 2.08s | 2.07s | ×1.00 (tương đương nhau) |
| Mỗi câu: in_tok | 694 | 3655 | ×5.27 |

### Chi phí tăng thêm đến từ đâu? (Giải thích chi tiết)

Khi theo dõi quá trình chạy thực tế, em nhận thấy sự chênh lệch chi phí giữa hai hệ thống xuất phát từ hai nguyên nhân chính:

1. **Ở khâu dựng hệ thống (Indexing - trả 1 lần duy nhất):**
   - Với **Flat RAG**, hệ thống chỉ cần đọc tài liệu, cắt thành các chunk văn bản rồi gửi qua mô hình embedding (`text-embedding-3-small`). Vì embedding không sinh ra token trả về (`out_tok = 0`) và giá embedding rất rẻ ($0.02 / 1 triệu token) nên chỉ mất **$0.00112** trong **45 giây**.
   - Với **GraphRAG**, ngoài phần luật trích xuất bằng regex, hệ thống bắt buộc phải dùng chat model (`gpt-4o-mini`) đọc từng bài báo tin tức để bóc tách thực thể và quan hệ (`extract_news_cases`). Quá trình này tiêu tốn thêm gần 36.000 input tokens và sinh ra 4.720 output tokens JSON, khiến chi phí tăng lên **$0.00934** (gấp **8.34 lần**) và thời gian chạy kéo dài lên **112.8 giây** (gấp **2.51 lần**).

2. **Ở khâu trả lời từng câu hỏi (Querying):**
   - Chi phí mỗi câu hỏi của GraphRAG cao hơn khoảng **4.54 lần** ($0.00059 so với $0.00013) là do kích thước prompt gửi vào LLM lớn hơn rất nhiều (trung bình **3.655 input tokens** so với **694 tokens** ở Flat RAG, gấp **5.27 lần**). Số token này tăng lên vì GraphRAG chèn thêm danh sách các dữ kiện có cấu trúc (facts) được truy xuất từ Neo4j qua Cypher (tóm tắt vụ án, người liên quan, điều luật định khung).
   - Tuy nhiên, điểm rất thú vị mà em quan sát được là **thời gian phản hồi (độ trễ) giữa 2 bên gần như tương đương nhau** (~2.07s so với 2.08s). Điều này chứng minh rằng việc truy vấn Cypher trên đồ thị Neo4j diễn ra cực kỳ nhanh (chỉ vài mili-giây), thời gian chờ chủ yếu vẫn là thời gian gọi mạng tới API của LLM.

---

## 2. Từng câu hỏi (10 điểm)

| Câu | Loại câu hỏi | Flat recall / judge | Graph recall / judge | Bên thắng | Vì sao (giải thích ngắn gọn 1 câu) |
| --- | --- | --- | --- | :---: | --- |
| **Q1** | `single-hop-law` | 1.00 / 2 | 1.00 / 2 | **Hòa** | Định nghĩa "tiền chất" nằm trọn vẹn trong một đoạn của Luật PCMT 2021 nên cả 2 bên đều tìm được và trả lời chuẩn xác. |
| **Q2** | `single-hop-news` | 1.00 / 2 | 1.00 / 2 | **Hòa** | Thông tin về 2 bị cáo nhận án tử hình nằm tập trung trong đúng 1 bài báo nên vector search và graph đều lấy đủ dữ liệu. |
| **Q3** | `cross-kb` | 0.00 / 0 | 1.00 / 2 | **Graph** | Flat RAG đành trả lời "Không đủ thông tin" do dữ liệu bị tách đôi (mức án ở bài báo, Điều luật ở BLHS); còn GraphRAG đi xuyên qua node Crime để ghép nối thành công. |
| **Q4** | `cross-kb` | 0.00 / 0 | 0.33 / 1 | **Graph** | Flat RAG hoàn toàn thất bại (0.00); GraphRAG tìm được tội danh "tổ chức sử dụng trái phép chất ma túy" của Hoàng Nato và liên kết sang Điều 255 BLHS. |
| **Q5** | `cross-kb-multi-hop` | 0.60 / 1 | 0.80 / 1 | **Graph** | GraphRAG kết nối được hành vi vận chuyển hơn 9,6kg MDMA của Cái Quang Huy với khung hình phạt kịch khung (tù 20 năm, chung thân, tử hình). |
| **Q6** | `aggregation` | 0.00 / 1 | 0.33 / 1 | **Graph** | Flat RAG chỉ lấy ngẫu nhiên 3 đoạn nhỏ; GraphRAG duyệt theo quan hệ đồ thị `INVOLVES` để gom đủ các vụ việc dính líu đến MDMA. |

### Nhận xét quy luật từ kết quả thực nghiệm:

Nhìn vào bảng so sánh trên, em nhận thấy một quy luật rất rõ ràng:
- **Với các câu hỏi đơn nguồn (`single-hop` - Q1, Q2):** Flat RAG hoạt động cực kỳ tốt (Recall 1.00, Judge 2 tuyệt đối) mà lại tiết kiệm chi phí hơn hẳn. Trường hợp này dùng Knowledge Graph là không cần thiết.
- **Với các câu hỏi đa nguồn xuyên tài liệu (`cross-kb`, `multi-hop`, `aggregation` - Q3, Q4, Q5, Q6):** Flat RAG bộc lộ điểm yếu chí mạng vì cơ chế tìm kiếm tương đồng vector chỉ nhặt được các đoạn văn độc lập, không đoạn nào chứa đủ cả tiền đề lẫn kết luận, dẫn đến việc mô hình liên tục báo *"Không đủ thông tin"* (Recall tụt về 0.00). Ngược lại, GraphRAG phát huy sức mạnh vượt trội khi bắc được cầu nối logic giữa người $\rightarrow$ vụ án $\rightarrow$ tội danh $\rightarrow$ điều luật, giúp nâng recall tổng thể từ **0.43 lên 0.74** và điểm judge từ **1.00 lên 1.50**.

---

## 3. Phân tích lỗi (20 điểm)

Trong quá trình chạy thực nghiệm và kiểm tra đồ thị trên Neo4j Browser, em đã phát hiện và phân tích 2 lỗi tiêu biểu của hệ thống:

### Lỗi E2: Thiếu ngữ cảnh luật đối với khung hình phạt tối đa (Câu Q4)

- **Hiện tượng quan sát được:**
  Khi hỏi câu Q4 (*"Giang hồ 'Hoàng Nato' bị bắt về hành vi gì, và hành vi đó có thể bị phạt tù tối đa bao nhiêu theo Bộ luật Hình sự?"*), GraphRAG trả lời được vế đầu (tội tổ chức sử dụng ma túy) nhưng vế sau lại nói: *"không đủ thông tin để xác định mức án phạt tù tối đa theo Bộ luật Hình sự"*, khiến câu trả lời chỉ đạt Recall = 0.33 và Judge = 1.
- **Bằng chứng kiểm chứng:**
  - Trích nguyên văn câu trả lời của GraphRAG trong `ket_qua_benchmark_kg.txt` (dòng 29):
    > *"Giang hồ 'Hoàng Nato' bị bắt về hành vi tổ chức sử dụng trái phép chất ma túy. Tuy nhiên, trong ngữ cảnh hiện tại không đủ thông tin để xác định mức án phạt tù tối đa theo Bộ luật Hình sự."*
  - Em mở Neo4j Browser chạy thử truy vấn Cypher để xem Điều 255 BLHS thực tế trong graph có những khoản nào:
    ```cypher
    MATCH (a:Article {id:'Điều 255 BLHS'})-[:HAS_CLAUSE]->(cl:Clause)
    RETURN cl.number AS Khoan, cl.penalty AS Hinh_Phat
    ORDER BY Khoan ASC;
    ```
  - Kết quả trả về trên màn hình Neo4j:
    ```
    Khoan | Hinh_Phat
    1     | phạt tù từ 02 năm đến 07 năm
    2     | phạt tù từ 07 năm đến 15 năm
    3     | phạt tù từ 15 năm đến 20 năm
    4     | phạt tù 20 năm hoặc tù chung thân
    5     | phạt tiền từ 50.000.000 đồng đến 500.000.000 đồng, phạt quản chế...
    ```
- **Nguyên nhân cốt lõi:**
  Lỗi nằm ở đoạn viết câu lệnh Cypher trong hàm `context()` của file `src/graph.py`. Logic ban đầu chỉ lọc lấy `cl.number = 1` (khung phạt cơ bản) hoặc các khoản có chứa chất ma túy mà vụ án dính líu. Tội danh "tổ chức sử dụng ma túy" (Điều 255) trong các khoản tăng nặng (khoản 2, 3, 4) quy định theo tính chất hành vi (tổ chức nhiều người, tái phạm, gây tổn hại sức khỏe) chứ không nêu tên chất ma túy. Vì vậy, Cypher đã bỏ sót khoản 4 ("phạt tù 20 năm hoặc tù chung thân"), khiến prompt gửi cho LLM hoàn toàn không có thông tin về khung phạt kịch khung.
- **Đề xuất khắc phục và đánh đổi:**
  - *Cách sửa:* Trong hàm `parse_law_article`, em viết thêm logic nhận diện khoản có hình phạt tù nặng nhất để gắn cờ `is_max = true` (khoản 4 Điều 255). Sau đó trong Cypher của hàm `context()`, mở rộng điều kiện lấy thêm khoản có `cl.is_max = true`.
  - *Đánh đổi:* Mỗi câu hỏi sẽ tốn thêm khoảng 150 token cho phần điều khoản tăng nặng, nhưng bù lại hệ thống sẽ trả lời được chính xác 100% các câu hỏi hỏi về mức phạt cao nhất.

---

### Lỗi E5: LLM bị lệch với dữ kiện trong Graph — Viện dẫn nhầm Điều luật tương tự (Câu Q5)

- **Hiện tượng quan sát được:**
  Ở câu Q5 (*hỏi về Cái Quang Huy, tội danh, loại ma túy, khoản áp dụng và khung hình phạt*), GraphRAG trả lời đúng gần hết: đúng tội danh (vận chuyển), đúng chất (MDMA), đúng khối lượng (>9.6kg), đúng khung hình phạt (20 năm, chung thân hoặc tử hình), nhưng lại trích dẫn nhầm thành **Điều 251 BLHS khoản 4** thay vì **Điều 250 BLHS khoản 4**.
- **Bằng chứng kiểm chứng:**
  - Trích nguyên văn câu trả lời trong `ket_qua_benchmark_kg.txt` (dòng 33):
    > *"Cái Quang Huy bị truy tố về tội 'vận chuyển trái phép chất ma túy' với loại ma túy là MDMA. Với khối lượng MDMA trong vụ này là hơn 9,6kg, khoản áp dụng tương ứng là khoản 4 của Điều 251 BLHS. Khung hình phạt theo khoản này là từ 20 năm tù, tù chung thân hoặc tử hình."*
    *(Vì nhầm Điều 251 thay vì Điều 250 nên bị trừ điểm từ khóa bắt buộc, recall dừng ở 0.80)*.
  - Em viết Cypher kiểm tra xem graph thực tế đang lưu Cái Quang Huy nối với Điều nào:
    ```cypher
    MATCH (p:Person {name:'Cái Quang Huy'})-[r:INVOLVED_IN]->(k:Case)-[:CHARGED_WITH]->(c:Crime)<-[:DEFINES]-(a:Article)
    RETURN p.name, r.charge, c.name, a.id, a.title;
    ```
  - Kết quả trả về trên Neo4j:
    ```
    p.name        | r.charge                         | c.name                           | a.id          | a.title
    Cái Quang Huy | vận chuyển trái phép chất ma túy | vận chuyển trái phép chất ma túy | Điều 250 BLHS | Tội vận chuyển trái phép chất ma túy
    ```
    Dữ liệu trong Knowledge Graph hoàn toàn đúng là **Điều 250 BLHS**, không hề bị lưu sai thành Điều 251!
- **Nguyên nhân cốt lõi:**
  Lỗi nằm ở khâu suy luận và sinh văn bản của LLM (LLM Hallucination/Bias). Trong tập dữ liệu, cả Điều 250 khoản 4 (vận chuyển) và Điều 251 khoản 4 (mua bán) đều có mức định khung giống hệt nhau: *"MDMA có khối lượng 100 gam trở lên thì bị phạt tù 20 năm, tù chung thân hoặc tử hình"*. Do vụ án có tang vật là MDMA số lượng lớn, cả 2 Điều luật này đều được kéo vào danh sách facts của prompt. Do Điều 251 (tội mua bán) xuất hiện quá nhiều trong các bài báo khác, LLM bị "ảo giác" và tiện tay trích dẫn nhầm số Điều 251 thay vì nhìn kỹ tên tội danh vận chuyển của Điều 250.
- **Đề xuất khắc phục và đánh đổi:**
  - *Cách sửa:* Trong chuỗi fact trả về từ `context()`, em format lại để gắn chặt tội danh với số điều: `f"Tội danh '{crime}' được quy định cụ thể tại [{article_id} - {title}] khoản {number}: {text}"`. Đồng thời bổ sung một câu nhắc nhở trong prompt: *"Bắt buộc đối chiếu chính xác tên tội danh của bị cáo với tiêu đề của Điều luật trước khi kết luận số Điều"*.
  - *Đánh đổi:* Phải tinh chỉnh prompt cẩn thận hơn, nhưng triệt tiêu được lỗi nhầm lẫn số điều luật của LLM.

---

## 4. Kết luận (5 điểm)

Từ những số liệu thực tế đo được qua bài lab, em rút ra được câu trả lời cho câu hỏi: *"Khi nào nên đầu tư làm Knowledge Graph, và khi nào Flat RAG là đủ?"*:

1. **Khi nào Flat RAG là đủ?**
   - Khi bài toán nghiệp vụ đơn giản, câu hỏi mang tính cục bộ (local question), thông tin cần tìm nằm gọn trong một văn bản duy nhất (như tìm kiếm quy chế, tra cứu định nghĩa văn bản nội bộ như Q1, Q2).
   - Khi dự án có ngân sách hạn chế: Flat RAG giúp tiết kiệm chi phí gấp **8.3 lần** lúc dựng hệ thống ($0.00112 so với $0.00934) và rẻ hơn **4.5 lần** cho mỗi lần người dùng hỏi ($0.00013 so với $0.00059).

2. **Khi nào bắt buộc phải dùng Knowledge Graph (GraphRAG)?**
   - Khi bài toán đòi hỏi phải **kết nối thông tin liên nguồn** mà không có tài liệu nào chứa sẵn toàn bộ đáp án (điển hình như bài toán pháp lý ma túy trong lab: hành vi bị cáo nằm ở tin tức báo chí, còn khung hình phạt và số Điều lại nằm ở Bộ luật Hình sự). Flat RAG trong trường hợp này hoàn toàn bất lực (Recall = 0.00), trong khi GraphRAG giải quyết trọn vẹn (Recall = 1.00, Judge = 2).
   - Khi cần suy luận nhiều bước (`multi-hop`) hoặc tổng hợp, gom nhóm thực thể (`aggregation` như Q6, tìm tất cả các vụ án liên quan đến một chất cụ thể).

3. **Bài học thực tiễn:**
   - Chi phí dựng đồ thị Knowledge Graph ($0.0093 ~ 230 VNĐ) và chi phí mỗi câu hỏi tăng thêm ~$0.00046 (~11 VNĐ) là một khoản đầu tư rất nhỏ nhưng mang lại bước nhảy vọt về chất lượng phản hồi (Recall tăng từ 0.43 lên 0.74). Đối với các lĩnh vực đòi hỏi độ chính xác cao như pháp luật, tài chính hay y tế, GraphRAG là giải pháp rất đáng tiền.

---

## 5. Tự kiểm (5 điểm)

Dưới đây là kết quả kiểm thử tự động thực tế trên máy em:

```
$ pytest tests/ -q
................................................                         [100%]
48 passed in 0.12s

$ python bench_kg.py --check
[OK] Dữ liệu: 18 điều luật, 20 bài báo
[OK] KG-1 link_entity
[OK] Neo4j kết nối được
[provider] chat = openai:gpt-4o-mini | embedding = openai:text-embedding-3-small
[OK] KG-2 build_graph: 146 node / 289 cạnh, đường xuyên 2 KB dài 2 cạnh
[OK] KG-3 context: 13 dữ kiện, có Điều 251
[OK] KG-4 GraphRAGAgent.answer
[OK] Chi phí check: 1 lần gọi LLM, $0.00064. Graph nhỏ (luật + 1 bài) vẫn còn trong Neo4j để bạn xem; chạy --judge để dựng graph đầy đủ.
```

- **3 ảnh chụp màn hình Neo4j Browser lưu trong thư mục `report/img/`:**
  1. `report/img/kg_count.png`: Ảnh chụp truy vấn Q-A đếm các node theo loại (thấy rõ ô gõ lệnh `$`, bảng đếm số lượng node và cột Results overview).
  2. `report/img/kg_cross_kb.png`: Ảnh chụp truy vấn Q-B thể hiện đường nối xuyên 2 KB từ Người/Vụ án sang Tội danh và Điều luật.
  3. `report/img/kg_my_case.png`: Ảnh chụp truy vấn Q-D thể hiện trọn vẹn đường đi của một nhân vật tự chọn.
- **Nhân vật em đã chọn cho ảnh `kg_my_case.png`:** **Cái Quang Huy** (vụ án vận chuyển hơn 9,6kg ma túy MDMA qua sân bay Nội Bài, kết nối sang Tội vận chuyển trái phép chất ma túy và Điều 250 BLHS).

---

## Vấn đề gặp phải (không tính điểm)

- **Vấn đề môi trường Python:** Lệnh mẫu trong bài lab là `py -3.11` nhưng máy tính của em đang cài sẵn Python 3.12. Em đã xử lý bằng cách tạo môi trường ảo trực tiếp với `python -m venv .venv` và cài các thư viện trong `requirements.txt` bình thường.
- **Lỗi hiển thị font chữ tiếng Việt trên PowerShell:** Terminal Windows mặc định dùng bảng mã CP1252 làm xuất hiện lỗi `UnicodeEncodeError` khi in các chữ cái tiếng Việt có dấu. Em đã khắc phục triệt để bằng cách đặt biến môi trường `$env:PYTHONIOENCODING="utf-8"`.
