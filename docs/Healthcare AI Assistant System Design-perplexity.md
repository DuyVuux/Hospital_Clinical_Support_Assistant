# Architecture for a Hospital Clinical Support Assistant

## 1. Problem Breakdown

The assistant must support three primary capability areas: summarizing patient medical records, guiding internal workflows, and answering administrative questions, all under strict healthcare regulations and integration with legacy hospital systems. Clinical use cases include generating longitudinal patient summaries, visit note digests, medication overviews, and highlighting guideline-relevant issues to assist (not replace) clinician judgment. Administrative use cases include explaining SOPs, routing staff through step-by-step workflows, answering benefits and billing questions, and helping front-desk staff with scheduling and paperwork.[1][2][3][4][5][6]

### 1.1 Core use cases

Clinical-facing use cases (higher risk):

- Longitudinal patient summary from EMR: condense problem lists, encounters, labs, imaging, and medications into a structured snapshot for clinicians.
- Visit/encounter summary: summarize last note or current-visit documentation for handoff between clinicians and nurses.
- Medication overview: list active meds, doses, dates, interactions to support med reconciliation (with clear caveats that it is not prescribing).
- Guideline and hospital protocol lookup: retrieve local clinical pathways, order sets, and checklists relevant to a disease or procedure.
- Pre-round briefings: generate patient-by-patient summaries for a ward round from latest vitals, notes, and orders.

Administrative-facing use cases (lower risk):

- SOP Q&A: natural-language questions over hospital policies, infection control procedures, documentation requirements, escalation rules, etc.[1]
- Workflow guidance: interactive checklists (e.g., how to admit a patient, discharge workflow, pre-op checklist, lab specimen routing) driven by SOP corpus.
- Forms assistant: explain which administrative or consent form is needed and pre-fill non-clinical parts from existing data for staff verification.
- Scheduling assistant: answer questions about clinic hours, appointment types, basic triage rules, and find open slots using scheduling APIs.
- Coding and billing support: help staff map documentation to codes (ICD, CPT) with human coder verification before submission.[2]

### 1.2 Risk levels by use case

Healthcare AI safety literature emphasizes task-specific risk assessment for LLM tools used in clinical contexts. A practical categorization for this system:[7][3][6]

- **High-risk (requires strict guardrails and mandatory human review):** any output that might influence diagnosis, treatment choice, medication changes, or high-stakes triage (e.g., suggesting different dose, proposing a new diagnosis); guideline interpretation that could be mistaken as personalized orders; summarization that might omit critical information used for decisions.[5][6][7]
- **Medium-risk:** clinical context understanding (e.g., summarizing history for handoff), highlighting possible issues (e.g., potential drug duplication) but clearly labeled as “for review only,” and internal workflow guidance that could affect timeliness and correctness of clinical tasks.
- **Low-risk:** administrative Q&A over policies, navigation of SOPs, pure informational support, and schedule or logistics questions that do not directly alter medical decisions.[4][2]

The system should enforce different UX patterns per risk level: high-risk flows require explicit confirmation that a clinician is responsible, display source citations, and often require dual sign-off; lower-risk flows can be more conversational.

## 2. High-Level Architecture

The system is a multi-tier application with a secure frontend, stateless backend APIs, a dedicated AI orchestration layer, a data plane for RAG and logs, and an integration layer that connects to HIS/EMR and other hospital systems.[8][2][4]

### 2.1 Textual system diagram

Describe the logical flow as layers:

- **Client layer (Web/mobile/desktop):** role-aware UI for doctors, nurses, and front-desk staff.
- **API gateway:** single entry point enforcing authN/Z, rate limiting, and request routing.
- **Application services layer:** domain-specific microservices (Patient Summary Service, Workflow Guidance Service, Admin Q&A Service, User Profile Service).
- **AI orchestration layer:** orchestrator service that performs RAG, prompt assembly, model routing, and post-processing.
- **Knowledge and vector store layer:** document storage for SOPs, policies, and de-identified artifacts plus a vector DB with hybrid search indexes.[9][10][11]
- **Integration layer:** connectors for EMR/HIS (FHIR/HL7 APIs), scheduling, identity (IdP), and logging/monitoring.
- **Security and compliance services:** audit logging, PHI tokenization/de-tokenization service, key management, and access-policy enforcement.[12][13][4][8]

A typical request path:

1. User authenticates via hospital SSO (e.g., SAML/OIDC) and accesses the assistant UI.
2. UI sends request to API gateway with JWT including role, department, and tenant identifiers.
3. Gateway authorizes and forwards to the appropriate application service.
4. Service calls integration layer (e.g., FHIR API) to gather structured data for the patient, enforcing scope via RBAC/ABAC.
5. Service calls AI orchestration layer with:
   - Structured context (e.g., vitals, labs) as JSON.
   - Unstructured context references (e.g., note IDs, SOP doc IDs) to retrieve via RAG.
6. Orchestrator performs retrieval, builds prompts, routes to chosen model, and returns answer plus citations and intermediate reasoning metadata.
7. Application service enforces risk-level UX (e.g., requiring clinician confirmation) and returns response to UI.
8. Audit service logs key fields (user, patient, docs used, model, risk level, prompt IDs) without storing more PHI than necessary.

### 2.2 Sync vs async flows

- **Synchronous (online) flows:** conversational Q&A, per-patient summary, SOP lookup, simple scheduling queries; require sub-3–5 second responses, so they use fast retrieval and inference with strict timeouts.[11]
- **Asynchronous flows:** batch ward-round packs (dozens of patients), bulk SOP re-indexing, nightly embeddings refresh, offline evaluation jobs; can run through a queue and notify the user when ready.

## 3. AI System Design

### 3.1 Model strategy

Regulatory and privacy guidance strongly favor HIPAA-compliant deployment models, either via on-premise or BAA-backed cloud services for PHI. A practical strategy:[2][12][4][8]

- **Primary LLM (clinical/complex tasks):** a high-quality model hosted in a HIPAA-compliant environment (e.g., Azure OpenAI with BAA, Vertex AI with BAA, or a strong open model like Llama‑3 or Mistral deployed in a secure VPC or on-prem cluster).[12][4][8][2]
- **Secondary LLM (administrative/light tasks):** a smaller, cheaper model (e.g., Phi‑3, Gemma, Mixtral or similar) for low-risk admin Q&A and simple rewriting, either self-hosted or via compliant API.[4]
- **Specialized smaller models:**
  - NER model for PHI detection/redaction.
  - Clinical entity extractor (problems, meds, allergies) using a biomedical model.
  - Classification models for task type, risk level, and routing.

Routing logic:

- Use a lightweight classifier to tag each query as clinical-high-risk, clinical-medium, or admin-low-risk.
- Clinical-high-risk → large clinical-grade LLM + strict RAG + mandatory sources + explicit disclaimers.
- Clinical-medium → large or medium model with RAG, but simpler constraints.
- Admin-low-risk → small model with RAG over SOP and policy corpus.

### 3.2 RAG architecture

**Vector DB choice:** For healthcare-grade deployments, a self-hosted or VPC-managed vector DB (e.g., PostgreSQL with pgvector, OpenSearch/Elasticsearch with k-NN, or a dedicated engine such as Weaviate or Qdrant) avoids sending PHI to third-party SaaS. The important properties are:[14][10][9][11]

- Support for hybrid dense + sparse search.
- Per-tenant and per-collection isolation.
- Efficient k-NN on encrypted disks with role-aware filters.

**Chunking strategy:**

- SOPs/workflows: chunk by logical steps or headings (e.g., 300–800 tokens), preserving section titles and breadcrumb path (department, policy ID).
- Clinical documents for RAG (if used): prefer de-identified or minimally necessary segments; chunk notes by section (HPI, labs, plan) or daily note, not arbitrary windows.
- Metadata: include document type, department, validity dates, approval version, and patient/encounter IDs when needed.

**Retrieval strategy:**

- Use **hybrid search**: combine BM25 (lexical) with vector similarity and fuse rankings via Reciprocal Rank Fusion (RRF).[10][9][14][11]
- First-stage retrieval: top‑20 candidates from both dense and sparse search in parallel with 300–500 ms timeout per backend.[11]
- Second-stage **cross-encoder re-ranking**: use a smaller cross-encoder (e.g., clinical sentence transformer) to re-rank top‑20 to top‑5 passages fed to the LLM.[9][10]
- Hard filters by department, patient ID, and policy validity; no document from outside the user’s allowed scope should be retrieved even if semantically similar.

### 3.3 Prompt design strategy

Prompts must:

- Explicitly state the assistant’s role (supporting clinician, not making decisions).
- Require citing specific source passages and returning answer + list of used document IDs and line numbers.
- Instruct the model to answer “I don’t know” or “No relevant policy found” when the retrieved context is insufficient.[7][1]
- Enforce format: for clinical summary, e.g., sections for Problems, Medications, Allergies, Recent Events, Outstanding Issues.
- Reflect risk level: high-risk prompts always remind that final decisions rest with the clinician and that the output may be incomplete.

For administrative flows, prompts can be simpler but still require quoting policy text and pointing to SOP sections.

### 3.4 Guardrails and hallucination control

Recent work on medical LLMs emphasizes safety protocols: constrained scope, strong retrieval, and structured human oversight. Guardrails include:[3][6][1][7]

- **Grounding requirement:** always supply retrieved context and instruct model to only answer from that context; if needed, show citations in UI.
- **Answerability check:** use a classifier or a secondary model to check whether the answer is grounded in the retrieved text (e.g., entailment between answer and sources).
- **Safety filters:** post-process outputs to flag forbidden behaviors (e.g., prescribing, dose calculation) and either block, soften, or route for review.
- **Role-limited capabilities:** for front-desk, suppress any interpretive clinical answers; only allow administrative content.
- **Template-based responses for high-risk areas:** for example, medication-related suggestions are phrased as “Potential issues for clinician review include …” rather than direct orders.
- **Benchmarked models:** evaluate models against clinical safety benchmarks (e.g., med-safety-benchmark, Clean & Clear) before deployment.[6][7]

## 4. Security and Compliance

### 4.1 Data encryption

HIPAA-compliant AI deployment requires encryption for PHI at rest and in transit with strong key management.[13][8][12][4]

- **In transit:** enforce TLS 1.2+ with modern ciphers for all internal and external service calls, including EMR APIs and model endpoints.
- **At rest:** encrypted volumes for databases, vector stores, logs, and model storage (e.g., AES‑256); enable TDE for SQL databases.
- **Key management:** use an HSM-backed KMS (e.g., AWS KMS, Azure Key Vault, GCP KMS, or on-prem HSM) for keys, with separation of duties; rotate keys regularly.
- **Field-level encryption or tokenization:** wrap PHI fields (patient name, MRN, identifiers) using a tokenization service; store tokens in logs instead of raw PHI.[13][12]

### 4.2 Access control (RBAC/ABAC)

HIPAA guidance emphasizes strict access control, minimal necessary access, and strong authentication.[8][4][13]

- **RBAC:** roles such as Attending Physician, Resident, Nurse, Front-Desk, Billing; each mapped to precise permissions (e.g., which patients, which tasks, which data fields).
- **ABAC:** additional attributes such as department, ward, shift, location (onsite vs remote), and relationship with patient (direct care vs not) to further restrict access.
- **Integration with IdP:** use SSO with SAML/OIDC to obtain user identity and roles; propagate as signed JWT across services.
- **Context-aware rules:** e.g., front-desk can see schedule and demographic data but not detailed clinical notes; nurses can see patients on their ward; only physicians can access certain summary features.

### 4.3 Audit logging

HIPAA requires detailed access logs for PHI, including who accessed what and when, and AI-focused guidance recommends logging prompt/response metadata as well.[12][4][13][8]

- Log: user ID, role, patient ID, documents retrieved (IDs only), model used, timestamps, risk level, and whether human approval was recorded.
- Store only minimal necessary text snippets in logs and avoid whole prompts or responses if they include PHI; instead, keep references (IDs) and hashed values.
- Protect audit logs with separate encryption keys and stricter access policies (security and compliance teams only).
- Implement tamper-evident logging (append-only or WORM storage) for regulatory audits.

### 4.4 PHI handling and data isolation

To maintain privacy, providers often keep all PHI inside controlled environments and avoid sending raw PHI to generic LLM APIs.[2][4][13][8][12]

- **Internal-only PHI:** store and process PHI inside hospital-controlled infrastructure (on-prem or VPC with BAA); if using external LLM APIs, ensure:
  - Provider signs a BAA.
  - Data is not used for training.
  - Region and residency align with regulations.
- **Tokenization/redaction:** PHI in prompts can be replaced with reversible tokens before calling a model; answers are then de-tokenized for display.
- **Tenant isolation:** for multi-hospital deployments, separate per-tenant databases and vector indexes; do not cross-contaminate data or embeddings.
- **Department and role isolation:** use metadata filters when retrieving from vector DB so that only documents from the user’s allowed departments are visible.

### 4.5 On-prem vs cloud vs hybrid

Recent guidance notes two common patterns for HIPAA-compliant LLMs: fully internal models and cloud-hosted compliant platforms under BAAs.[4][13][8][2][12]

- **On-prem:**
  - Pros: maximum control, keeps all PHI on hospital infrastructure, easier to satisfy strict national regulations.
  - Cons: requires significant infra and MLOps expertise; hardware and upgrade costs; slower access to latest frontier models.
- **Cloud (HIPAA-compliant PaaS):**
  - Pros: managed security, easier scaling, access to advanced models; major providers (Azure, GCP) offer HIPAA-ready LLM services with BAAs.[13][4]
  - Cons: data residency and vendor lock-in concerns; need to carefully configure VPC peering, private endpoints, and logging.
- **Hybrid (recommended for many hospitals):**
  - PHI and core systems (EMR, vector DB, orchestration, logs) reside in on-prem or hospital-controlled VPC.
  - LLM endpoints are in cloud with BAA and called via private connectivity with tokenized prompts.

The choice depends on local regulations and hospital IT maturity; the hybrid pattern offers a pragmatic balance between security and capability for many institutions.

## 5. Integration with Legacy Systems

### 5.1 Integration patterns

Hospitals often use HL7 v2 messaging, proprietary APIs, and increasingly FHIR for modern integrations.[8][4]

- **FHIR APIs:** use FHIR resources (Patient, Encounter, Observation, MedicationRequest, Condition, DocumentReference) for structured data ingestion when available.
- **HL7 v2:** interface engine subscribes to ADT, ORU, ORM, and other messages; normalize to internal data models for the assistant.
- **Proprietary APIs/DB access:** for legacy systems without standards, use read-only views or integration-engine adapters to produce canonical JSON.

### 5.2 Data sync strategy

- **Real-time:** for per-request summaries, fetch the latest data on demand from EMR via FHIR/HL7 interfaces; optionally cache read-only copies for performance with short TTL.
- **Near-real-time streaming:** use an integration engine to push new notes, labs, and orders into a staging store and, when appropriate, into the RAG index (with de-identification where needed).
- **Batch:** off-hours jobs to refresh full SOP corpus, training data, and embeddings; nightly or hourly incremental re-indexing.

### 5.3 Failure handling when legacy is down

- Implement circuit breakers and health checks on integration connectors.
- If EMR is unavailable:
  - Degrade gracefully by limiting the assistant to SOP and policy Q&A only.
  - Display clear UI banners: “Clinical data currently unavailable; only policy and admin support are active.”
  - Queue non-urgent tasks and re-run when connectivity is restored.

## 6. Cost Architecture

### 6.1 Cost components

- **LLM costs:** token-based usage for primary and secondary models; dependent on prompt length, retrieved context, and response size.[4]
- **Infrastructure:** compute for orchestration services, vector DB, integration engine, and, if self-hosting, GPU/CPU clusters; storage for logs, embeddings, and document stores.[2][12]
- **Hidden costs:** logging and observability (APM, tracing), security tooling, backups, evaluation pipelines, and human review time for high-risk queries.[7][12]

### 6.2 MVP vs scale (order-of-magnitude reasoning)

Public HIPAA-ready LLM deployments and internal LLM reports highlight that token and infra costs scale roughly linearly with usage but can be reduced by routing and caching. A practical planning approach:[12][2][4]

- **MVP (single hospital, limited pilots):**
  - Users: 50–100 clinicians/staff.
  - Daily queries: 1,000–2,000.
  - Average tokens per query (prompt + context + response): 2,000–3,000.
  - LLM usage: a few million tokens per day, on the order of tens to low hundreds of USD per day for API-based models, or equivalent GPU time if self-hosted.
  - Infra: small Kubernetes cluster (or equivalent VMs), 1–2 TB encrypted storage for vector DB and logs.
- **5× scale:** roughly 5× traffic and storage; costs scale proportionally unless mitigated by caching and smaller models.
- **10× scale:** at this point, it becomes important to:
  - Aggressively route admin queries to small models.
  - Use semantic caching to avoid recomputation of repeated questions.
  - Consider self-hosting models if utilization is high enough to justify GPU capex.

Exact numbers depend on chosen vendor pricing and local infra costs, but the architecture is designed so that each component’s cost can be measured and optimized independently.

## 7. Optimization Strategy

### 7.1 Model routing

- Use a cheap classifier/small LLM to categorize requests (clinical vs admin; high vs medium vs low risk).
- Route low-risk admin queries to smaller models; reserve expensive clinical-grade model for complex clinical context.
- Optionally, route certain template-like tasks (e.g., converting notes to summaries with fixed schema) to fine-tuned small models.

### 7.2 Caching

- **Semantic cache:** store embeddings and answers for common questions (e.g., “How to admit a patient to ward X?”); use similarity search over past queries to reuse validated answers.
- **Per-patient cache:** cache computed summaries for a patient for a short window (e.g., a shift), invalidated when new notes or orders arrive.
- **HTTP and application-level caching:** for static SOP documents and policy text.

### 7.3 Prompt optimization

- Minimize prompt length by:
  - Restricting context to top‑k passages and structured snippets instead of entire documents.
  - Using compressed context representations (summaries of SOP sections) where safe.[14][10][9][11]
- Use consistent templates so that tokens can be predicted and controlled.
- Monitor token usage per query type and adjust k, chunk sizes, and response format to fit budget.

### 7.4 When to switch to smaller/self-hosted models

- If clinical-heavy traffic is modest but admin traffic is high, keep primary clinical model as managed API and move admin traffic to self-hosted small models.
- If total token volume crosses a threshold where dedicated GPU clusters are cheaper than API, migrate to self-hosted models inside hospital or VPC, keeping the same orchestration interface.
- For very strict jurisdictions, prefer on-prem/self-hosted models to avoid cross-border data concerns.[2][12]

## 8. Scaling and Reliability

### 8.1 Handling traffic spikes and latency

Production RAG systems benefit from autoscaling, parallel retrieval, and careful timeouts.[10][9][14][11]

- **Autoscaling:** container orchestration (e.g., Kubernetes) with HPA on CPU, memory, and queue length for AI orchestration and vector DB.
- **Parallel retrieval:** hybrid dense + sparse retrieval executed in parallel with independent timeouts (e.g., 500 ms each), using whichever returns first; RRF fusion when both succeed.[11]
- **Timeouts and fallbacks:** if retrieval or model inference exceeds SLAs, respond with partial results, a shorter summary, or ask the user to refine the query.

### 8.2 Queue system, retries, circuit breakers

- **Queue system:** use a message queue (e.g., RabbitMQ, Kafka, or cloud equivalent) for asynchronous jobs (batch summaries, re-indexing); worker services consume and process them.
- **Retries:** exponential backoff with idempotent job design; cap retries to avoid runaway cascades.
- **Circuit breakers:** monitor downstream systems (EMR, LLM endpoints, vector DB); open circuits when failures exceed thresholds, and degrade functionality (e.g., policy-only mode) while showing clear user feedback.

### 8.3 Fallback strategies

- **Simpler model:** on LLM provider failure, route admin queries to an alternate small model (self-hosted or from another compliant provider).
- **Rule-based system:** for critical workflows (e.g., triage, sepsis alerts), maintain existing rule-based CDSS; the assistant augments with explanations but does not replace these safeguards.[5]
- **Human escalation:** when confidence is low, retrieval fails, or safety filters trigger, respond with “Unable to provide a reliable answer; please consult a senior clinician or specialist,” and optionally route to a human support channel.

### 8.4 Key monitoring metrics

- Technical: request rate, latency percentiles, error rates per service, token usage, cache hit rates.
- Clinical/operational: number of clinical queries vs admin, frequency of overridden suggestions, flagged unsafe outputs, time saved on documentation or workflow steps.[1][7]
- Security/compliance: access denials, unusual access patterns, failed logins, and log integrity checks.[13][8][12][4]

## 9. Evaluation and Accuracy Strategy

### 9.1 Measuring correctness

For medical LLMs, typical evaluation pipelines include domain-specific benchmarks and real-world case reviews.[3][6][5][7]

- Use established clinical safety and QA benchmarks (e.g., med-safety-benchmark, Clean & Clear) to test models’ propensity to hallucinate or omit critical details.[6][7]
- Build an internal evaluation set of de-identified patient cases and SOP questions with gold-standard answers