# Sprint 1 Proposal – Trợ lý lâm sàng AI cho bệnh viện

## 1. Project overview
- **Giải pháp:** Xây dựng trợ lý lâm sàng AI tích hợp sâu với HIS/EMR/scheduling để hỗ trợ ba luồng chính: tóm tắt lịch sử bệnh nhân (high-risk), hướng dẫn SOP/quy trình nội bộ (medium-risk) và trả lời câu hỏi hành chính/lịch hẹn (low-risk).  
- **Kiến trúc tổng thể:** Frontend role-aware (SMART on FHIR, SSO/OIDC, human-in-the-loop UI) → API Gateway (authn/authorize, RBAC/ABAC, rate limit) → Service orchestration (intent classifier, model router, risk label) → AI layer gồm RAG pipeline (chunking theo section HPI/Assessment/Plan + hybrid dense+sparse search), vector DB (namespace isolation Qdrant/Weaviate tự host) và LLM (GPT-4/GPT-4o hoặc frontier cho lâm sàng, GPT-3.5/LlaMA/Phi3 cho admin). Lớp dữ liệu xử lý connector HL7/FHIR (HL7 engine, SMART, webhooks), OCR cho tài liệu giấy, preprocessing de-ID + embedding.
- **Luồng dữ liệu:** Người dùng gửi truy vấn, orchestrator xác định intent và mức rủi ro, RAG truy xuất các chunk phù hợp (tuỳ theo patient ID, role), prompt có cấu trúc kèm yêu cầu trích dẫn. Sau inference, cấu phần guardrail kiểm tra PHI, đánh dấu hallucination, gửi câu trả lời kèm citation về frontend để clinician xác nhận.

## 2. Enterprise context
- **Khách hàng:** Bệnh viện lớn tại khu vực phải tuân thủ HIPAA/GDPR, có hệ sinh thái EMR legacy (HL7 v2, FHIR, data warehouse, lịch đặt, SOP dạng PDF). Người dùng bao gồm bác sĩ, y tá, lễ tân, coder/billing, mỗi vai trò có quyền truy cập khác nhau.  
- **Dữ liệu:** ePHI (hồ sơ bệnh án, kết quả xét nghiệm, thuốc), SOP, lịch khám, form hành chính. Xử lý yêu cầu chunk metadata chứa patient ID, department, version SOP, quyền.  
- **Ràng buộc enterprise:** mã hóa AES-256 at rest và TLS 1.3 in transit; RBAC/ABAC kết hợp (vai trò + context shift/ward/department) để hạn chế truy cập; audit logging immutable (hash chain), log PHI token hóa, giữ ít nhất 6 năm; phải có BAA với mọi đối tác xử lý PHI; đồng thời đảm bảo vectordb namespace ngăn tenant/department lan tràn.
- **Tính chất legacy:** cần HL7 engine (Mirth Connect) xử lý ADT/ORU, convert thành FHIR, có queue (Kafka/RabbitMQ) để đệm khi EMR offline; fallback message queue cho data ingest và replay khi có lỗi.

## 3. Deployment choice
- **Chọn Hybrid Cloud:** Dữ liệu nhạy cảm (EMR, vector index, tokenized PHI) nằm on-prem hoặc trong VPC riêng của bệnh viện để đảm bảo data sovereignty; inference nặng (LLM GPT-4, re-ranking) chạy trên HIPAA-eligible cloud (Azure/AWS/Azure OpenAI Health, Anthropic) qua private endpoint/VPN và tokenized prompts.  
- **Lợi ích:** Giữ PHI trong ranh giới hospital-controlled, tận dụng elastic compute cloud cho inference, dễ scale model routing và multi-region backup.  
- **Cơ chế:** On-prem chịu trách nhiệm cho connector HL7/FHIR, preprocessing, vector DB, guardrail. Cloud xử lý inference (LLM calls: GPT-4o/Azure, fallback secondary provider), logging (encrypted CloudWatch) với truy vấn không chứa raw PHI (chỉ tokenized IDs).  
- **Recovery:** Thiết lập multi-cloud fallback, circuit breaker nếu provider lỗi. Khi cloud không khả dụng, hệ thống degrade xuống self-hosted LLM (Llama3/Mistral) và rule-based search để tiếp tục hỗ trợ admin.

## 4. Cost analysis (MVP + Growth)
- **MVP (0–3 tháng):**  
  - *Infrastructure:* 1 node vector DB (4 CPU / 16 GB RAM ~ $15/tháng), API/orchestrator nhỏ, ELK/CloudWatch minimal logging.  
  - *AI:* dùng GPT-3.5 hoặc LLM nhỏ (Phi3, LLaMA 2) cho admin/light tasks và GPT-4 (Azure OpenAI) cho lâm sàng high-risk. Ước tính 200 truy vấn lâm sàng + 300 admin mỗi ngày → mỗi tóm tắt 5–10K token (prompt+response) ~ $0.15–$0.30/truy vấn → ~$30–60/ngày. Chi phí token ~ $1K–$2K/tháng, cộng storage + logging + monitoring ~ $5K–$10K/tháng (bao gồm đội ngũ SRE, security, human review).  
  - *Nhân sự & hidden cost:* human-in-loop review, compliance audit, security testing và document policies; chi phí training, documentation, HIPAA audit.  
- **Growth (5×–10×):**  
  - *Scaling vector:* cần thêm nodes (60GB RAM, nhiều CPU), có thể $130+/tháng cho RAM; metadata indexing phức tạp.  
  - *AI cost:* token/ngày tăng 5–10×, cần semantic cache, model routing để chuyển các truy vấn low/medium risk sang LLM nhỏ tự host (Llama3 8B, Mixtral) giảm token spend.  
  - *Compute:* GPU server on-prem nếu chuyển inference high-volume, đầu tư vài ngàn USD.  
  - *Chi phí khác:* tăng monitoring (thêm dashboard, alert) và nhân lực (SRE/Obs, compliance, HITL reviewers).  
  - *Cân nhắc:* khi token cost quá cao, ưu tiên self-host hoặc hybrid inference, giữ AI expensive chỉ cho high-risk.

## 5. Optimization plan
- **Model routing:** classifier xác định risk (high/medium/low) dựa trên intent, metadata, và user role. High-risk → GPT-4/GPT-4o hoặc medical frontier model; medium → GPT-4 Mini/GPT-3.5; admin low-risk → Llama3/Phi3. Mỗi route kèm metadata để track model/cost.  
- **Semantic cache:** lưu snapshot kết quả phổ biến (VD: tóm tắt A lặp lại trong 5 phút) với expire trigger khi nguồn cập nhật, giúp giảm inference token spend và latency.  
- **Chunking & retrieval:** chunk theo section (HPI, Assessment, Plan) và SOP heading; hybrid search (dense vector + BM25) với RRF, re-ranking bằng cross-encoder, hạn chế chunk theo patient, department.  
- **Prompt design:** persona rõ ràng, cấu trúc sections (Problems, Medications, Allergies, Tasks), yêu cầu trích dẫn từng đoạn (document ID, line). Gắn prompt “If data insufficient, respond ‘Không đủ thông tin’.”  
- **Guardrails:** NeMo guardrails, uncertainty score detection, PHI filter (NER, tokenization), output validation so sánh với retrieved doc, block nếu hallucinates. Human-in-loop review buộc với high-risk responses (clinician phải approve).  
- **Cải tiến:** A/B test prompt templates, measure groundedness metric, track model latency/cost, pipeline retraining prompts from HITL feedback.

## 6. Reliability plan
- **Circuit breaker & fallback:** monitoring failure rate của LLM provider; nếu vượt threshold, vòng breaker mở và chuyển sang secondary provider (Anthropic/Azure). Nếu cả hai down, degrade xuống self-hosted lightweight LLM (Llama3/Mistral) hoặc deterministic Q&A.  
- **Queue & priority:** sử dụng Kafka/RabbitMQ để xử lý async (batch summaries, embeddings rebuild). Định nghĩa priority queue (emergency ward > outpatient scheduling). Retry/backoff policy cho connectors (HL7/EMR).  
- **Monitoring & alert:** track TTFT, RAG accuracy, cache hit, error rate, hallucination count, token cost, latency. Alerts cho drift, high error, cost spike. Dashboards + logging (immutable) cho audit (user, patient, docs).  
- **Deployment:** blue-green / canary rollout, versioned prompts/model configs. Rollback với single-click nếu metrics degrade.  
- **Audit & logging:** log user, patient ID (hash), documents retrieved, model, risk level, timestamp; logs encrypted, tamper-evident.  
- **Fallback SOP:** When real-time data offline, show warning “Không thể đồng bộ dữ liệu EMR. Dùng thông tin cache gần nhất và kiểm tra lại.”  

## 7. Track recommendation + next step
- **Track Phase 2:** Bắt đầu với use case administrative/SOP Q&A + scheduling để giảm rủi ro. Triển khai HL7-to-FHIR sync read-only, đánh giá UX với người dùng admin, thu feedback từ human-in-loop.  
- **Next steps:**  
  1. Hoàn thiện connector HL7/FHIR + metadata chunk, đảm bảo data tagging theo department và version.  
  2. Xây semantic cache & guardrail framework (tokenization, citation, NeMo).  
  3. Cài đặt dashboard giám sát KPIs (accuracy, cost, latency, guardrail violations).  
  4. Chuẩn bị CI/CD + Kubernetes deployment blue-green/canary cho backend/RAG.  
  5. Thu thập feedback HITL, làm A/B test prompt + model routing trước khi mở rộng summarization lâm sàng.
