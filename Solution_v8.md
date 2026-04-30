# Exhaustive AI-Driven Contract Analysis & ERP Integration Solution
**Date:** 2026-04-27
**Subject:** Comprehensive Technical Solution Proposal for Tariff Change Management

## Assignment Coverage Map

| Assignment Requirement | Section |
|---|---|
| **Task 1 — Solution Architecture** | |
| Core system components | [§1.1 Core AI Components](#11-core-ai-components--responsibilities) |
| AI/agent usage | [§1.1](#11-core-ai-components--responsibilities), [§7 Tradeoffs](#7-key-architectural-tradeoffs--ai-boundary-constraints), [§A.1 Agentic Framework](#a1-agentic-framework--infrastructure-aws-strands--agentcore) |
| Data flow and integration points | [§1.2 Data Payload Lifecycle](#12-data-payload-lifecycle--flow), [§3 Integration Strategy](#3-integration-strategy) |
| Human-in-the-loop touchpoints | [§1.1](#11-core-ai-components--responsibilities), [§2.2 Autonomy Lifecycle](#22-the-autonomy-progression-lifecycle-production) |
| Security and governance | [§5 Security, Access & Governance](#5-security-access--governance) |
| **Task 2 — AI Usage Design** | |
| Where AI is used | [§1.1](#11-core-ai-components--responsibilities), [§7](#7-key-architectural-tradeoffs--ai-boundary-constraints) |
| Where AI is NOT used | [§7 — financial arithmetic, template selection, residency routing](#7-key-architectural-tradeoffs--ai-boundary-constraints) |
| How autonomy levels are determined | [§2.2 Autonomy Progression](#22-the-autonomy-progression-lifecycle-production) |
| Where human review is required | [§1.1 — three named HITL touchpoints](#11-core-ai-components--responsibilities) |
| How risk is mitigated | [§1.3 Schema Enforcement](#13-forcing-probabilistic-ai-into-deterministic-outcomes), [§6 Failure Modes](#6-failure-modes--system-recovery) |
| **Task 3 — Integration Strategy** | |
| Enterprise data sources | [§3 — PMIS, ERP, document repositories](#3-integration-strategy) |
| Workflow systems | [§3 — Step Functions, EventBridge, PMIS handoff](#3-integration-strategy) |
| Action systems (notifications, document generation) | [§1.1 Change Notice Drafting Agent](#11-core-ai-components--responsibilities), [§3](#3-integration-strategy) |
| **Task 4 — Observability & Operations** | |
| Logging and telemetry | [§4.2 Observability Tools](#42-observability-tools), [§5.3 Audit Governance](#53-audit-governance) |
| AI performance monitoring | [§2.1 Evaluation Framework](#21-pre-production-ai-evaluation-the-evaluation-framework) |
| Failure handling and fallbacks | [§6 Failure Modes](#6-failure-modes--system-recovery) |
| Scaling strategy | [§4.1 System Scaling](#41-system-scaling--load-leveling) |
| **Task 5 — Delivery Considerations** | |
| Key risks | [§8.2 Key Risks](#82-key-risks) |
| Phased rollout strategy | [§8.1 Phased Rollout](#81-phased-rollout-strategy) |
| Technical dependencies | [§8.3 Technical Dependencies](#83-technical-dependencies) |
| **Scenario Requirements** | |
| Detect contract change signals | [§1.1 — Textract + Comprehend NER pipeline](#11-core-ai-components--responsibilities) |
| Draft change notices | [§1.1 Change Notice Drafting Agent](#11-core-ai-components--responsibilities), [§3](#3-integration-strategy) |
| Quantify schedule and cost impacts | [§1.1 Impact Assessment Agents](#11-core-ai-components--responsibilities) |
| Track change and claim lifecycle | [§1.1 Claim Lifecycle Tracker](#11-core-ai-components--responsibilities), [§1.2 step 8](#12-data-payload-lifecycle--flow) |
| Integrate with PMIS, ERP, document repositories | [§3 Integration Strategy](#3-integration-strategy) |
| Support human review and escalation | [§1.1](#11-core-ai-components--responsibilities), [§5.2 Action Governance](#52-action-governance) |
| Incomplete and inconsistent data | [§1.4 Edge Cases](#14-handling-edge-cases-incomplete--inconsistent-data) |
| Legal concerns about AI-drafted outputs | [§1.1 — "AI drafts; human owns and signs"](#11-core-ai-components--responsibilities) |
| Multiple enterprise systems | [§3 — modern API + legacy batch + MCP gateway](#3-integration-strategy) |
| Governance and compliance | [§5 Security, Access & Governance](#5-security-access--governance) |
| Mixed levels of AI autonomy | [§2.2 Autonomy Progression](#22-the-autonomy-progression-lifecycle-production) |

---

## 1. Solution Architecture & Granular Workflows

This system utilizes a highly specialized, agentic workflow designed to automate the detection, extraction, and quantification of tariff changes within unstructured and semi-structured contracts, integrating securely with legacy PMIS and ERP systems.

### 1.1 Core AI Components & Responsibilities
* **Ingestion Layer (AWS Textract):** Responsible for the initial OCR, document structure analysis, and raw text conversion of all incoming contract documents. *(Note: Textract relies on traditional deep learning and computer vision, not Generative AI, providing highly stable, deterministic text extraction).* A **PII detection pass (Amazon Comprehend PII)** is run immediately after Textract extraction — sensitive entities (names, signatures, bank details) are tagged and redacted from LLM-facing payloads by default, with access restored only at the UI layer based on role.
* **Extraction & Entity Recognition (Amazon Comprehend):** Utilizes a custom-trained entity recognizer (traditional NLP Named Entity Recognition, not GenAI) specific to legal and corporate terminology to extract critical metadata, specifically the **Contract ID**, **Vendor Name**, and specific line items.
* **Inference & Lineage Mapping (AWS Strands & Bedrock Models):** When explicit lineage is missing from the document, this agent infers the missing project data based on historical context, attempting to link the isolated Contract ID to specific project IDs.
* **Impact Assessment Agents (Cost & Schedule):** A specialized set of AI agents (designed with strict separation of concerns) dedicated to assessing the exact impact of the contract changes.
* **Change Notice Drafting Agent (AWS Strands + Template Repository):** After impact is quantified, a dedicated agent retrieves the applicable change notice template from a structured template repository (S3 or ProjectWise/SharePoint). Template selection is **deterministic** — driven by contract type, jurisdiction, and counterparty attributes from the project master record in PMIS, not by semantic similarity. The agent populates the template with extracted change data and produces a structured draft surfaced to the Human Review Portal for redlining. **AI drafts; the human commercial manager owns and signs the final outbound document.** This directly addresses the legal concern about AI-generated outputs.
* **Claim Lifecycle Tracker (AWS Step Functions + PMIS Polling):** A Step Functions state machine tracks the system's view of each change through its lifecycle states: *Detected → Assessed → Notice Drafted → Submitted → Acknowledged → Disputed → Negotiated → Settled → Closed.* At the point of ERP/PMIS handoff, the state machine registers a task token and uses scheduled EventBridge polling to observe state transitions in the downstream PMIS workflow. The Step Function is an **observer, not the system of record** — PMIS owns the claim lifecycle; this layer provides visibility and triggers downstream actions (notifications, escalations) based on state changes. Polling is idempotent; if a state transition is missed due to downtime, the next poll cycle catches up. This architecture is explicitly eventually consistent, not real-time.

### 1.2 Data Payload Lifecycle & Flow
Understanding how the data shape transforms at each hop is critical to the system's integrity.
1.  **Raw Input:** Unstructured PDF or Scanned Image (Binary).
2.  **Textract Output:** Standardized JSON containing hierarchical text blocks, key-value pairs, and bounding box coordinates.
3.  **PII Tagging:** Comprehend PII detection runs immediately — sensitive entities are tagged and redacted from all downstream LLM-facing payloads. Role-based access controls govern visibility at the UI layer.
4.  **Comprehend NER Output:** Enriched JSON payload. The raw text is appended with structured metadata arrays (e.g., `[{"ENTITY": "VENDOR", "VALUE": "Acme Corp", "CONFIDENCE": 0.98}]`).
5.  **Agentic Inference:** The JSON payload is mutated to include inferred lineage.
6.  **Rules Engine / Impact Agent Output:** The enriched JSON is mapped to a strict, rigid schema required by the target ERP system (e.g., `{"erp_contract_id": "12345", "cost_delta": 500.00, "unit": "LBS"}`).
7.  **Change Notice Draft Output:** The impact payload, combined with a deterministically selected template, is rendered into a populated change notice draft — surfaced to the Human Review Portal for redlining before any outbound submission.
8.  **Claim Lifecycle Event:** On ERP/PMIS push, a Step Functions task token is registered and the claim enters the lifecycle tracking state machine (*Detected → Assessed → Drafted → Submitted → Acknowledged → Disputed → Negotiated → Settled → Closed*).

### 1.3 Forcing Probabilistic AI into Deterministic Outcomes
Generative AI produces probabilistic, text-based outputs. Enterprise ERP systems require strict, deterministic inputs. We bridge this gap through a **Schema Enforcement Pipeline**:
* **Strict Prompting:** The agents are instructed to output *only* valid JSON matching a predefined schema, with no conversational filler.
* **Schema Validation (Pydantic / JSON Schema):** Immediately after the LLM generates a response, the payload is passed through a deterministic validation layer. If the payload is missing a required key or contains a string instead of a float, the validation fails.
* **Deterministic Fallback:** If validation fails, the system automatically attempts one retry with the error message fed back to the LLM. If it fails again, it deterministically routes to the Human Review queue. No malformed or probabilistic data ever reaches the ERP integration layer.

### 1.4 Handling Edge Cases: Incomplete & Inconsistent Data
* **Inconsistent Data (e.g., Unit Conversions & Baseline Lag):** If a contract utilizes one unit of measure (e.g., "Tons") but the target ERP requires another (e.g., "Pounds"), the agent compares the extracted unit against the standard ERP unit; if a mismatch occurs, it pauses the workflow and explicitly flags the item for human conversion.
* **Incomplete Data Handling:** If critical metadata is missing, the system immediately halts processing for that document and routes to a human reviewer.

---

## 2. AI Usage Design, Evaluation, & Managed Autonomy

### 2.1 Pre-Production AI Evaluation: The Evaluation Framework

What is described here is not a one-time pre-production check but a **continuous quality system** with stratified benchmarking at each pipeline stage, explicit production thresholds, and a feedback loop that keeps the benchmark current as contract language evolves.

**Stratified Golden Dataset**

We derive a Golden Dataset from 5,000 to 10,000 historical contracts where the final, approved ERP entries are already known and verified. Critically, the dataset is **stratified** — not randomly sampled — to ensure coverage across:
* Contract type (subcontract, vendor supply, professional services)
* Jurisdiction and language
* Data completeness (clean contracts vs. those with missing lineage — the hard cases)
* Change complexity (simple unit price changes vs. multi-clause amendments)

A dataset weighted toward clean, simple contracts produces a misleading F1 score. Hard cases are what breaks production systems and must be explicitly represented.

**Layered Metrics Per Pipeline Stage**

A single system-level F1 score is insufficient. We measure a distinct metric at each pipeline hop:

| Stage | Metric | Rationale |
|---|---|---|
| Textract | Character Error Rate | Validates OCR accuracy on raw text |
| Comprehend NER | Entity-level Precision & Recall (separate) | A missed claim (false negative) is worse than a flagged false positive — recall weighted higher |
| Lineage Inference | % correct project ID match | Validates RAG + SQL join accuracy |
| Final ERP Payload | Exact match rate vs. verified historical entry | End-to-end ground truth check |

**Explicit Production Thresholds**

* **Extraction F1 > 0.92** before Phase 1 HITL deployment
* **ERP payload exact match > 0.85** before any autonomy is granted
* **Human override rate < 5%** sustained over 30 days before reducing HITL volume

The asymmetry is intentional: extraction noise is caught by humans downstream; a malformed ERP payload is expensive and sometimes irreversible to correct.

**Continuous Evaluation via Production Feedback Loop**

Every human override in production is a new labeled example. When a reviewer corrects the AI's extraction, the original document, the AI output, and the human correction are written to a **Feedback Store**. Periodically, a data engineer promotes high-confidence corrections into the golden dataset. This provides two capabilities:
* **Drift Detection:** If new contract language is systematically harder than the baseline, the live override rate climbs before the static dataset F1 shows any degradation — giving early warning of distribution shift.
* **Continuous Model Improvement:** Comprehend custom entities and Bedrock prompts are tuned against real production failures, not just historical data.

### 2.2 The Autonomy Progression Lifecycle (Production)
1.  **Phase 1: 100% Human-in-the-Loop (HITL).** Initially, the system operates with zero autonomy. 
2.  **Phase 2: Metric Tracking ("Mismatched Reviews").** The system meticulously tracks instances where the human reviewer overrides or corrects the AI's proposed extraction.
3.  **Phase 3: Threshold Trigger.** Once the rate of human overrides drops below the **5% threshold**, the system's escalation rules are dynamically tweaked.
4.  **Phase 4: Increased Autonomy.** The volume of escalations kicked out to humans is systematically reduced.

---

## 3. Integration Strategy

* **Modern API Integration:** Utilizes **Model Context Protocol (MCP)** servers and custom agents via AgentCore Gateway.
* **Legacy System Integration:** Relies on scheduled **batch integrations** and automated file generation for heritage mainframes.
* **PMIS Integration (e.g., Primavera P6, Microsoft Project):** Schedule impact data is written back to PMIS via API where available, or via structured file export (XER/XML) for systems that do not support direct API integration. The claim lifecycle Step Function polls the PMIS workflow API on a scheduled EventBridge cadence to track state transitions. Polling is idempotent and eventually consistent — not real-time.
* **Document Repository Integration (ProjectWise / SharePoint):** Change notice templates are retrieved from the document repository at draft generation time. Approved, signed change notices are written back to the repository as part of the HITL approval step, maintaining a single source of truth for contract correspondence.
* **Template Selection:** Template retrieval is deterministic — driven by a structured lookup against contract type, jurisdiction, and counterparty attributes from the PMIS project master record. This is not a semantic search operation.

---

## 4. Observability, Telemetry, & Scaling Strategies

**Core Assumption:** The system is explicitly designed as a **Batch Mode, Asynchronous, Event-Driven** architecture. It does not process synchronous REST requests.

### 4.1 System Scaling & Load Leveling
Because the architecture decouples ingestion from processing via message queues (AWS SQS), massive volume spikes (e.g., end-of-year contract renewals) will not crash the system. 
* **Queue Depth Scaling:** CloudWatch monitors the `ApproximateNumberOfMessagesVisible` metric in SQS. As the queue deepens, AWS Application Auto Scaling incrementally spins up additional Lambda concurrency or ECS tasks for the Impact Agents.
* **The HITL Scaling Bottleneck:** A critical operational reality is that **the process can only scale as fast as humans can review the exceptions.** Even if the AI processes 10,000 contracts in a minute, a 5% exception rate means 500 documents hit the Human Review Portal. SLA times must be managed based on workforce capacity, not just compute capacity.

### 4.2 Observability Tools
* **AgentCore Observability & AWS CloudWatch:** Macro observability tracking queue depths, latency, and human override rates.
* **OpenTelemetry (OTel):** Micro-tracing documenting the exact path a request takes.
* **Deployment Hash:** Every log entry and OTel trace is injected with a Deployment Hash (`commit_sha` and `model_version`) for strict traceability.

---

## 5. Security, Access & Governance

Security and governance are structured across three distinct layers:

### 5.1 Data Governance
* **Data Residency:** Residency is determined at the point of human submission, not inferred from document content. The uploading user selects or confirms the project context at upload; the system inherits the data residency requirement from the **project master record in PMIS** and routes the document to the appropriate AWS region accordingly. AI never makes this determination. This handles international projects (UK, Middle East, Australia) without requiring document-level inference.
* **PII Handling:** A Comprehend PII detection pass runs immediately after Textract extraction. Sensitive entities (individual names, signatures, bank details) are tagged and redacted from all LLM-facing payloads at the processing layer — not just masked at the UI. Role-based access controls govern PII visibility at the storage and display layers. Users who lack PII access receive masked values; the underlying records retain the originals for authorized roles.
* **Encryption:** Encrypted at rest (AWS KMS) and in transit (TLS 1.2+).

### 5.2 Action Governance
* **Agent Identity & Access (AgentCore Identity):** Secure, scalable agent identity and access management integrating with AWS and third-party tools.
* **Role-Based Access Control (RBAC):** Tier 1 Analysts handle extraction corrections; Tier 2 Managers handle final ERP push approvals.
* **Threshold-Based Escalation:** Escalation is triggered by **change value as a percentage of contract value** (not absolute dollar amounts), with a floor absolute dollar threshold. Both the contract value and the threshold bands are inherited from the project master record in PMIS — not hardcoded. This allows thresholds to vary by client, contract type, and jurisdiction without code changes. Schedule impact equivalents (e.g., float consumption as a percentage of critical path) follow the same pattern.

### 5.3 Audit Governance
* **Operational Logs (OTel / CloudTrail):** Every log entry and OTel trace is injected with a Deployment Hash (`commit_sha` and `model_version`) for strict traceability. CloudTrail records all API-level actions.
* **Immutable Audit Records (Legal Admissibility):** Distinct from operational logs, a write-once audit record is maintained in S3 with **Object Lock (WORM)** enabled. Each record captures: who submitted the document, the raw AI extraction output, every human correction made, who approved, and precise timestamps for each state transition. This record is human-readable and reconstructible for use in arbitration or legal proceedings — answering definitively what the AI produced versus what the human approved.
* **Tamper-Evidence:** Audit records use hash chaining to ensure no entry can be silently modified after the fact.

---

## 6. Failure Modes & System Recovery

* **Failure 1: Bedrock / Textract API Throttling.** *Mitigation:* Asynchronous SQS queues hold the message with exponential backoff.
* **Failure 2: Legacy ERP System is Down.** *Mitigation:* Batch payloads are queued in staging tables until the system recovers.
* **Failure 3: Model Hallucination Spike.** *Mitigation:* A "Circuit Breaker" trips if override rates spike, returning the system to 100% HITL.

---

## 7. Key Architectural Tradeoffs & AI Boundary Constraints

* **Where AI Should NOT Be Used:** We explicitly separate semantic reasoning from mathematical computation. **Generative AI (LLMs) is never used for financial arithmetic, final cost aggregations, or unit conversions.** These are deterministic tasks executed by strict Python code within the rules engine (or via AgentCore Code Interpreter). Generative AI is reserved strictly for unstructured data extraction and lineage reasoning.
* **Traditional ML vs. Generative AI:** We use specialized, traditional ML models (Textract for Computer Vision, Comprehend for NLP NER) for initial passes to save token costs and latency, reserving expensive GenAI only for complex lineage mapping.
* **Risk vs. Latency:** Managed Autonomy trades immediate processing latency for high accuracy and zero business risk.

---

## 8. Delivery Considerations & Phased Rollout

### 8.1 Phased Rollout Strategy

**Phase 1 — Shadow Mode (Weeks 1–8)**
* Single contract type (e.g., subcontracts), single pilot project, single AWS region.
* System runs in full shadow mode: AI processes every document but all outputs go to human review with zero autonomy.
* Golden Dataset evaluation framework established; baseline F1 and override rates recorded.
* HITL portal and redlining workflow validated with actual reviewers.
* Gate criterion: Extraction F1 > 0.92 and ERP payload exact match > 0.85 on the stratified golden dataset.

**Phase 2 — Controlled Autonomy (Weeks 9–20)**
* Expand to additional contract types and a second project/region.
* Override rate monitoring begins. Autonomy thresholds remain locked until the 5% override rate is sustained over 30 days.
* Change notice drafting workflow goes live; legal review sign-off on AI-assisted output process obtained.
* PMIS polling and claim lifecycle tracking validated end-to-end.
* Gate criterion: 30-day sustained override rate < 5% on Phase 1 contract type.

**Phase 3 — Scaled Deployment (Week 21+)**
* Autonomy progressively increased on qualifying document classes.
* Multi-region routing and data residency controls fully activated.
* Feedback store loop operational — production corrections feeding back into model improvement cycle.
* Circuit breaker thresholds tuned based on observed production variance.

### 8.2 Key Risks

* **PMIS API Availability:** Legacy PMIS systems may not expose stable APIs. Mitigation: validate API contracts early in Phase 1; design file-based fallback for schedule data exchange from the outset.
* **Legal Sign-Off on AI-Assisted Change Notices:** Legal and commercial teams may require extended review time before accepting AI-drafted outputs. Mitigation: involve legal stakeholders in Phase 1 shadow review; frame AI as a drafting assistant with mandatory human sign-off, not an autonomous author.
* **Golden Dataset Quality:** If historical ERP entries contain errors, the benchmark is corrupted. Mitigation: data quality audit of historical records before dataset construction; stratified sampling reviewed by a domain expert before use.
* **Human Reviewer Capacity:** At scale, the HITL bottleneck constrains throughput regardless of compute capacity. Mitigation: SLAs set against reviewer workforce capacity, not compute; Phase 3 autonomy expansion directly relieves this constraint.

### 8.3 Technical Dependencies

* PMIS API documentation and sandbox environment access (Phase 1)
* ProjectWise / SharePoint API access for template repository integration (Phase 1)
* Legal approval of change notice drafting workflow and redlining process (Phase 2 gate)
* AWS Bedrock AgentCore availability in target regions (validate for non-US deployments)
* Historical contract corpus with verified ERP ground truth for golden dataset construction



### A.1 Agentic Framework & Infrastructure (AWS Strands & AgentCore)
To process the complex reasoning required for impact assessment without maintaining massive bespoke infrastructure, the system utilizes the next-generation AWS Agentic Stack:
* **The Framework (AWS Strands Agents):** The Cost and Schedule agents are built using the open-source AWS Strands SDK. This lightweight, code-first Python framework natively manages the agentic loop (planning, reasoning, tool selection) and integrates seamlessly with Model Context Protocol (MCP) servers to fetch data.
* **The Infrastructure (Amazon Bedrock AgentCore):** The Strands agents are deployed directly to Amazon Bedrock AgentCore Runtime. This provides a secure, serverless microVM for execution, completely eliminating the need to manage ECS clusters or complex Lambda orchestration.
* **Deterministic Math via AgentCore Code Interpreter:** To enforce the constraint that LLMs must never perform financial math, the Strands agents leverage the built-in AgentCore Code Interpreter tool. When a unit conversion (e.g., Tons to Pounds) or total cost delta calculation is required, the agent writes and executes a Python script within a secure sandbox, returning a 100% deterministic mathematical outcome.
* **Tool Orchestration via AgentCore Gateway:** Legacy ERP APIs are converted into agent-compatible tools using the AgentCore Gateway, streamlining connectivity.

### A.2 RAG & Vector Database Integration (Lineage Explainability)
To enable the AI agents to accurately infer missing project lineage from historical context without compromising system performance or cost, the architecture implements a Retrieval-Augmented Generation (RAG) pattern using a "Thin Vector, Fat Payload" design.
* **Deterministic Text Preparation:** Generative AI is *not* used to parse the raw Textract JSON. Instead, a deterministic Python function extracts clean text, drops coordinate geometry, and chunks the document while injecting structural metadata.
* **The "Thin Vector" Index:** The vector database (e.g., Aurora pgvector) acts strictly as a secondary index. It stores only the vector embedding and a foreign key (`document_id`) linking back to the Iceberg table.
* **The Join:** The Strands agent retrieves the nearest-neighbor `document_id`s from the vector DB, performs a deterministic SQL lookup against the Iceberg table to retrieve the verified metadata, and synthesizes both into the final inference prompt.

### A.3 Deep Dive: The Textract-to-Comprehend Pipeline (Custom Approach)
Amazon Comprehend requires clean text for accurate NLP processing. Feeding it raw Textract JSON destroys semantic accuracy and exponentially inflates token costs.
* **The Parsing Pipeline:** A Python routine iterates through the Textract JSON output, extracts only `"BlockType": "LINE"`, and concatenates the text strings into a clean paragraph.
* **The NLP Integration:** This clean string is sent to Comprehend for entity extraction.
* **Spatial Re-Assembly via Character Offsets:** Comprehend returns entities with `BeginOffset` and `EndOffset` integers (e.g., characters 13 through 22). The Python script maps these character offsets back to the original Textract block IDs, retrieves the original proportional Bounding Box coordinates, and stores them in Iceberg. This allows the Human Review frontend to visually highlight the exact extracted entity on the original PDF scan.

### A.4 The Managed Alternative: Amazon Q (Quick) Apps
If the business objective is to prioritize speed-to-market and eliminate the frontend development required for a custom Human Review portal, the system can pivot the user interface to **Amazon Q (Quick)**.
* **Agentic Workspace:** Amazon Q provides a secure, out-of-the-box enterprise UI. Reviewers log in via IAM Identity Center.
* **Human-in-the-Loop Orchestration:** The Q App interfaces with the underlying Bedrock Knowledge Bases and Strands/AgentCore logic. Analysts review the proposed impacts natively in the Q interface and execute approvals, which trigger the AgentCore Gateway plugins to update the ERP.
* **Tradeoff:** This drastically reduces UI development time but requires adapting the Human Review workflow to fit within the Amazon Q App's conversational interface paradigm.

## Notes
The ideas and architecture are my own. AI was used in the following ways in creation of this repo. 
* **Sparring Partner:** AI was used as a architect assistant to discuss pros and cons of each of the approaches and architectural decisions. 
* **Writing assistant:** AI was used to consolidate the discussion and create this md file.
