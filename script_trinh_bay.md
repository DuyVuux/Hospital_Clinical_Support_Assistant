# Script Trình Bày — ClinicalAI: Hospital Clinical Support Assistant
**Nhóm C401-F2 | Thời gian: 5 phút | Zone Presentation**

> 💡 **Cách dùng file này:**
> - Chữ thường = nói bình thường
> - **[IN ĐẬM]** = nhấn mạnh, nói chậm lại
> - *(nghiêng)* = gợi ý hành động / chuyển màn hình
> - `--- chuyển slide ---` = bấm sang slide tiếp theo
> - ⏱ = mốc thời gian tham khảo

---

## 🎤 SLIDE 1 — Project Overview
⏱ *0:00 – 0:45 (45 giây)*

*(mở đầu, nhìn thẳng vào audience)*

"Xin chào mọi người. Nhóm mình — nhóm C401-F2 — chọn **Scenario 2: Hospital Clinical Support Assistant.**

Bài toán là thế này: tại một bệnh viện đa khoa 500 giường, mỗi ca làm việc, bác sĩ và điều dưỡng mất trung bình **[15 đến 20 phút]** chỉ để... tra cứu quy trình nội bộ, tóm tắt hồ sơ bệnh nhân, hoặc trả lời những câu hỏi hành chính lặp đi lặp lại như 'giờ thăm bệnh là mấy giờ', 'cách thanh toán BHYT thế nào'.

Trong giờ cao điểm, điều đó gây tắc nghẽn toàn bộ quy trình.

Hệ thống mình đề xuất — **ClinicalAI** — giúp rút ngắn thời gian đó xuống còn khoảng **[30 giây].** Người dùng gồm ba nhóm chính: khoảng 80 bác sĩ, 150 điều dưỡng, và 70 nhân viên lễ tân — mỗi nhóm có nhu cầu và quyền truy cập khác nhau."

`--- chuyển slide ---`

---

## 🏢 SLIDE 2 — Enterprise Context & Constraints
⏱ *0:45 – 1:35 (50 giây)*

*(trỏ vào 3 card constraints)*

"Trước khi nói kiến trúc, mình cần làm rõ **bối cảnh enterprise** — vì nó quyết định toàn bộ các lựa chọn kỹ thuật phía sau.

Dữ liệu mà hệ thống đụng đến gồm bệnh án điện tử, kết quả xét nghiệm, chẩn đoán — đây là **[PHI — Protected Health Information]** — cực kỳ nhạy cảm. Theo Luật Khám bệnh chữa bệnh năm 2023, loại dữ liệu này **không được rời khỏi hạ tầng của bệnh viện.**

Ba ràng buộc lớn nhất mình xác định là:

Một — **tuân thủ pháp lý.** PHI bắt buộc ở on-prem. Không có ngoại lệ.

Hai — **tích hợp HIS legacy.** Bệnh viện đang chạy hệ thống cũ, thường dùng chuẩn HL7. Mình không thể yêu cầu họ migration toàn bộ.

Ba — **phân quyền nghiêm ngặt.** Bác sĩ chỉ thấy bệnh nhân của mình. Điều dưỡng không thấy chẩn đoán. Lễ tân chỉ thấy lịch hẹn.

*(trỏ vào dòng bảo mật phía dưới)*

Về bảo mật kỹ thuật: mã hóa **AES-256** at rest, **TLS 1.3** in transit, ký BAA với mọi cloud provider xử lý PHI, audit log immutable giữ tối thiểu **6 năm.**

Và điều quan trọng nhất — nếu AI trả lời sai thông tin lâm sàng, hậu quả ảnh hưởng trực tiếp đến bệnh nhân. Không giống các domain khác."

`--- chuyển slide ---`

---

## 🏗️ SLIDE 3 — Deployment Architecture
⏱ *1:35 – 2:35 (60 giây)*

*(trỏ vào toàn bộ diagram)*

"Từ những ràng buộc đó, mình chọn kiến trúc **Hybrid.**

*(trỏ vào vùng bên trái — On-prem)*

Vùng on-prem chịu trách nhiệm toàn bộ phần nhạy cảm. Luồng dữ liệu đi như này: HIS legacy gửi message HL7, qua **[Mirth Connect]** — đây là HL7 integration engine phổ biến nhất trong healthcare — convert sang FHIR rồi đẩy vào pipeline. **[Kafka queue]** đứng ở giữa để buffer và replay dữ liệu khi EMR offline — bệnh viện offline là chuyện bình thường, không thể mất data.

Tiếp theo là Vector DB dùng Qdrant tự host, với **namespace isolation** theo department — bác sĩ khoa A không truy cập được vector của khoa B. RAG Engine dùng **hybrid search** — kết hợp dense vector và BM25, qua cross-encoder re-ranking để tăng độ chính xác. Audit log **immutable**, giữ 6 năm.

Phân quyền dùng **RBAC kết hợp ABAC** — không chỉ theo role mà còn theo context: ward, department, ca trực. Bác sĩ A không thấy bệnh nhân của bác sĩ B dù cùng khoa.

*(trỏ sang vùng bên phải — Cloud)*

Cloud chỉ xử lý LLM inference — Claude hoặc Azure OpenAI — qua **private endpoint.** Và quan trọng: prompt chỉ chứa **[tokenized data]**, không có thông tin định danh thô. PHI không bao giờ rời hạ tầng bệnh viện.

Mọi câu trả lời lâm sàng — thuốc, chẩn đoán, liều lượng — phải qua **human review gate** trước khi hiển thị cho bác sĩ."

`--- chuyển slide ---`

---

## 💰 SLIDE 4 — Cost Analysis
⏱ *2:35 – 3:20 (45 giây)*

*(trỏ vào bảng bên trái)*

"Về cost, mình bóc tách theo từng lớp thay vì chỉ nhìn con số tổng.

Ở mức **MVP** — 50 users, 750 request mỗi ngày — tổng cost khoảng **[426 đô một tháng].** Khi scale lên **Growth** — 300 users, 6.000 request mỗi ngày — lên khoảng **[1.660 đô.]**

*(trỏ vào dòng Human review — màu đỏ)*

Nhưng nhìn vào đây — **[cost driver lớn nhất không phải LLM API.]** Mà là **human review labor** — chiếm gần **47%** tổng chi phí ở giai đoạn MVP. 200 đô một tháng chỉ để review câu trả lời. Đây là hidden cost mà hầu hết khi estimate AI system đều bỏ qua.

*(trỏ vào chart bên phải)*

Nhìn vào chart stacked bar — khi scale lên Growth, phần màu đỏ — human review — vẫn chiếm tỷ trọng lớn nhất. Đó là vấn đề mình sẽ giải quyết ở slide tiếp theo.

Hidden cost khác cần nhớ: re-indexing mỗi khi SOP cập nhật theo quý, training nhân viên, và compliance audit hàng năm."

`--- chuyển slide ---`

---

## ⚙️ SLIDE 5 — Cost Optimization Plan
⏱ *3:20 – 4:00 (40 giây)*

*(trỏ lần lượt vào 3 card)*

"Để tối ưu, mình chọn **ba chiến lược** — có cân nhắc thời điểm áp dụng.

**Chiến lược một — Model Routing.** Áp dụng ngay từ MVP. Câu hỏi FAQ hành chính đơn giản đi qua **Claude Haiku** — rẻ hơn nhiều. Câu hỏi lâm sàng phức tạp mới dùng **Claude Sonnet.** Tiết kiệm khoảng **40% LLM cost.**

**Chiến lược hai — Semantic Caching.** Cũng áp dụng ngay. Cache những câu hỏi hay lặp lại — giờ thăm bệnh, cách đặt lịch, thủ tục BHYT. Cosine similarity trên 0.92 thì trả kết quả cache, không gọi LLM nữa. Tiết kiệm **25% LLM calls.** Lưu ý quan trọng: **không cache** câu hỏi lâm sàng — risk stale data quá cao.

**Chiến lược ba — Selective Human Review.** Để giai đoạn Growth. Thay vì review tất cả, chỉ review khi confidence score thấp hơn 0.75 hoặc thuộc category lâm sàng. Giảm từ 10% xuống còn 3-4% request cần review — tiết kiệm **50% human review cost.**

*(trỏ vào thanh tổng kết phía dưới)*

Kết quả sau optimization: **MVP giảm 34%**, Growth giảm 37% tổng chi phí."

`--- chuyển slide ---`

---

## 🛡️ SLIDE 6 — Reliability & Scaling Plan
⏱ *4:00 – 4:45 (45 giây)*

*(trỏ vào bảng failure scenarios)*

"Về reliability — mình thiết kế cho ba failure scenario thực tế nhất.

Traffic tăng đột biến mùa dịch — dùng **Kafka priority queue** ưu tiên bác sĩ trực, rate limit theo role, tắt tạm tính năng nặng như tóm tắt toàn bộ hồ sơ.

LLM provider down — có **circuit breaker** tự động switch sau 3 lần lỗi.

Hallucination lâm sàng — đây nguy hiểm nhất. Xử lý bằng **[NeMo Guardrails]**: PHI filter, uncertainty score detection, mọi câu trả lời lâm sàng phải kèm **citation trích nguồn** từng câu — nếu không có nguồn, không trả lời.

*(trỏ vào fallback chain)*

Fallback chain 4 lớp: **Claude/GPT-4o** là primary. Nếu lỗi, switch sang **Azure OpenAI/Anthropic** — secondary provider. Nếu cloud down hoàn toàn, chuyển về **Llama3/Mistral self-hosted** on-prem. Lớp cuối là **keyword search** kèm cảnh báo — 'Không thể đồng bộ EMR, vui lòng kiểm tra lại.'

*(trỏ vào SLA metrics)*

SLA nội bộ: p95 latency dưới 3 giây, error rate dưới 1%, uptime 99.5%."

`--- chuyển slide ---`

---

## 🚀 SLIDE 7 — Track Phase 2 & Next Steps
⏱ *4:45 – 5:00 (15 giây)*

*(nói nhanh, tự tin, kết thúc gọn)*

"Nhóm chọn **AI Engineering Track** cho Phase 2 — vì nhóm mạnh nhất ở RAG, evaluation framework và agent design.

Roadmap 8 tuần: Sprint 1 — production-ready RAG với streaming. Sprint 2 — monitoring dashboard và hallucination detection. Sprint 3 — prototype tích hợp HIS thực qua HL7 FHIR. Sprint 4 — load testing và validation.

Bước ngay sau hôm nay: **user interview** với một bệnh viện nhỏ để validate, và implement semantic caching từ codebase Lab 14.

Cảm ơn mọi người — nhóm mình sẵn sàng nhận câu hỏi."

---

---

# ❓ Q&A — Câu Hỏi Thường Gặp

> Đọc câu hỏi → dừng 1 giây → trả lời tự tin, ngắn gọn

---

**Q1: Tại sao không dùng full on-prem cho an toàn hơn?**

"Chi phí server để chạy LLM frontier on-prem rất cao — ước tính gấp 3 đến 4 lần so với cloud inference. Với Hybrid, PHI vẫn ở on-prem đúng theo quy định — chỉ prompt đã được tokenize mới lên cloud. An toàn về pháp lý, tối ưu về chi phí."

---

**Q2: Mirth Connect là gì? Tại sao cần?**

"Mirth Connect là HL7 integration engine phổ biến nhất trong healthcare. HIS legacy của bệnh viện thường nói ngôn ngữ HL7 v2 — Mirth Convert sang FHIR để RAG pipeline đọc được. Không có lớp này thì không integrate được với hệ thống cũ mà không cần migration toàn bộ — rất tốn kém và rủi ro."

---

**Q3: Tại sao cần Kafka? Queue đơn giản hơn không được sao?**

"Kafka cho phép **replay message** khi EMR offline — dữ liệu không bị mất. Với bệnh viện, downtime HIS là chuyện xảy ra thường xuyên nên cần buffer đủ mạnh. Ở MVP dùng RabbitMQ đơn giản hơn cũng được — khi volume lớn thì nâng lên Kafka."

---

**Q4: Confidence score để detect hallucination tính thế nào?**

"Kết hợp hai lớp: RAG retrieval score — cosine similarity giữa câu trả lời và source document — và NeMo Guardrails output validation, so sánh câu trả lời với retrieved chunk. Nếu có đoạn nào không có nguồn → flag human review. Mọi câu trả lời lâm sàng bắt buộc phải có citation document ID cụ thể."

---

**Q5: Cost 426 đô MVP nghe có vẻ thấp — thực tế có đúng không?**

"426 đô là chi phí API và infrastructure thuần. Nếu tính cả team — SRE, compliance officer, security audit — con số thực tế cho một bệnh viện đầy đủ có thể lên 5 đến 10 nghìn đô một tháng. Nhưng ROI vẫn rõ ràng: 300 nhân viên tiết kiệm 15 phút mỗi ca — tính ra hàng trăm giờ lao động mỗi tháng."

---

**Q6: Nếu bệnh viện nhỏ không có hạ tầng on-prem tốt?**

"Dùng Private Cloud như Azure Government hoặc VNG Cloud — vẫn đảm bảo data sovereignty, PHI không ra nước ngoài. Phù hợp với bệnh viện tuyến huyện hoặc phòng khám không có server riêng."

---

**Q7: RBAC và ABAC khác nhau thế nào, tại sao cần cả hai?**

"RBAC kiểm soát theo role — bác sĩ, điều dưỡng, lễ tân. ABAC thêm context — cùng là bác sĩ nhưng bác sĩ khoa A không thấy bệnh nhân khoa B, bác sĩ ca sáng không thấy ca chiều. Kết hợp cả hai mới đủ granular cho môi trường bệnh viện thực tế."

---

**Q8: Phase 2 chọn AI Engineering Track, vậy các kỹ năng Infra hay Business có thiếu không?**

"Nhóm tự chấm Infra ở mức 3 trên 5 — biết RAG nhưng chưa có kinh nghiệm HIS integration thực tế. Đó là lý do Sprint 3 tập trung vào HL7 FHIR prototype — để bù đắp đúng điểm yếu đó. Business và Product ở mức đủ để validate use case qua user interview."

---

# 📝 Ghi Nhớ Khi Trình Bày

| Việc nên làm | Việc nên tránh |
|---|---|
| Nói chậm ở các con số và tên kỹ thuật | Đọc nguyên văn từng chữ trên slide |
| Trỏ vào diagram khi giải thích architecture | Đứng yên không tương tác với slide |
| Dừng 1 giây sau mỗi ý chính | Nói liên tục không có điểm nghỉ |
| Nhìn audience khi nói phần tổng kết | Nhìn màn hình suốt |
| Dùng "mình" nhất quán | Mix "chúng tôi" và "mình" lộn xộn |
| Tự tin nhận câu hỏi: "Câu hỏi hay!" | Nói "Câu này mình chưa nghĩ đến" |
