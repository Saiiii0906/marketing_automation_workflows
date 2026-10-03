# VenturelyHub Marketing & Sales Automation
## Workflow Catalogue

This catalogue tracks all n8n workflows managed within the VenturelyHub Marketing & Sales Automation system.

---

### 1. Lead Capture - Error Handler

- **Repository Filename:** `lead_generation/Lead Capture - Error Handler.json`
- **Workflow Name:** `Lead Capture - Error Handler`
- **Live n8n Workflow ID:** `baY6Z22zHL4swGCj`
- **Active Status:** `INACTIVE` (`active: false`)
- **Purpose:** Standalone centralized error handling workflow. Triggered automatically by n8n when a workflow configured with `errorWorkflow: "baY6Z22zHL4swGCj"` encounters an unhandled execution error. Appends error details to an external log tab.
- **Dependencies:**
  - Invoked by: `Universal Lead Capture System` (`JKfUhTnVOYnY03h9`) via `settings.errorWorkflow`
- **Credential Requirements:**
  - Google Sheets OAuth2 API (`googleSheetsOAuth2Api`) — currently unresolved/missing in local n8n runtime
- **Target Resources:**
  - Spreadsheet ID: `1BBHhzCAZXfy274_rxbPAT4_-xlXmFC1ToOLVcqLWo2U` (Sheet: `Error Log`)
- **Current Status:** Imported, validated (100% node pass), inactive. Reconciled with live local n8n runtime.

---

### 2. Universal Lead Capture System

- **Repository Filename:** `lead_generation/Universal Lead Capture System.json`
- **Workflow Name:** `Universal Lead Capture System`
- **Live n8n Workflow ID:** `JKfUhTnVOYnY03h9`
- **Active Status:** `INACTIVE` (`active: false`)
- **Purpose:** Multi-layer inbound lead processing pipeline:
  1. Trigger Layer: Webhook endpoint (`POST /webhook/lead-capture`)
  2. Validation Layer: Validates email syntax (regex) and phone number presence
  3. Branching: Routes invalid submissions to an invalid leads store
  4. Data Cleaning: Normalizes phone formats and sanitizes text inputs
  5. ID Generation: Assigns unique identifier (`LEAD-YYYYMMDD-RAND4`)
  6. Storage Layer: Appends valid leads to legacy sheet store (`Lead Database`)
  7. Notification Layer: Sends HTML lead notification to `coo.venturelyhub@gmail.com`
  8. Auto-Reply: Autoresponder template (disabled by default)
- **Dependencies:**
  - Error Workflow: Points to `baY6Z22zHL4swGCj` (`Lead Capture - Error Handler`)
- **Credential Requirements:**
  - Google Sheets OAuth2 API (`googleSheetsOAuth2Api`) — currently unresolved/missing in local n8n runtime
  - Gmail OAuth2 (`gmailOAuth2`) — currently unresolved/missing in local n8n runtime
- **Target Resources:**
  - Spreadsheet ID: `1BBHhzCAZXfy274_rxbPAT4_-xlXmFC1ToOLVcqLWo2U` (Sheets: `Lead Database`, `Invalid Leads`, `Error Log`)
  - Recipient Email: `coo.venturelyhub@gmail.com`
- **Current Status:** Imported, validated (100% node pass), inactive. Error workflow dependency successfully linked to live Error Handler ID. Legacy storage layer noted as interim automation store per Master Execution Contract (Absolute CRM Rule).

---

### 3. VenturelyHub Prospect Intelligence

- **Repository Filename:** `01_prospect_intelligence/prospect_intelligence.json`
- **Workflow Name:** `VenturelyHub Prospect Intelligence`
- **Live n8n Workflow ID:** `vhProspectInt001`
- **Active Status:** `INACTIVE` (`active: false`)
- **Total Nodes:** 20 nodes (15 functional nodes + 5 sticky documentation notes)
- **Purpose:** Stateless prospect intelligence pipeline automating B2B prospect research, strategic qualification, and execution audit logging:
  1. Trigger: `POST /webhook/prospect-intelligence`
  2. Input Validation: Enforces required `organization_name`, sanitizes URLs/location, identifies test fixtures
  3. Factual Research (Gemini): Uses Google Search and domain context to extract structured factual profile (programs, technologies, activities, public contact roles) with strict source citation
  4. Factual Structuring: Parses, validates research density, activates fallback if sparse
  5. Strategic Fit Analysis (Ollama Qwen3 8B): Evaluates VenturelyHub relevance, fit level (`HIGH`, `MEDIUM`, `LOW`, `INSUFFICIENT_DATA`), maps matching services, recommends segment-specific offer, crafts value proposition, personalization angle, and anticipates objections
  6. Output Formatting: Assembles contract-compliant payload
  7. Sheet Payload Preparation: Extracts flattened summary columns, preserves complete `raw_output_json`, assigns unique `execution_id`
  8. Persistent Audit Append: Appends record to `VenturelyHub` Google Spreadsheet (`Prospect Intelligence Outputs` on success, `System Events` on validation failure) with non-blocking error handling (`continueRegularOutput`)
  9. Direct Caller Response: Returns complete structured intelligence JSON payload directly to the HTTP caller
- **Dependencies:**
  - Standalone pipeline; no external workflow dependencies
- **Credential Requirements & Verification Status:**
  - Google Gemini API (`googlePalmApi`): Credential ID `9pUlYyCqAwOgAvb8` (`tiwarivivek102006@gmail.com`) — `AUTHENTICATION_VERIFIED`
  - Local Ollama API: Zero-credential HTTP endpoint (`http://127.0.0.1:11434/api/chat`, model `qwen3:8b`) — `AUTHENTICATION_VERIFIED / NATIVE_LOCAL`
  - Google Sheets OAuth2 API (`googleSheetsOAuth2Api`): Credential ID `JQRGvtvkfjEPF9WK` (`srisaikirantambalkar@gmail.com`) — `AUTHENTICATION_VERIFIED`
  - Anthropic API: Removed. Replaced by local Ollama Qwen3 8B.
- **Target Resources:**
  - Webhook endpoint: `/webhook/prospect-intelligence`
  - Spreadsheet: `VenturelyHub` (ID: `15__ZAea7cXNS0U3sd-GzsTuZWSFMIgomM_EHPjUr5N4`)
  - Target Worksheets: `Prospect Intelligence Outputs`, `System Events` (and 6 future phase worksheets)
- **Current Status:** Implemented, 100% node validation pass, live runtime verified (Executions 7-12 verified across all 6 test scenarios with Qwen3 8B local inference and live Google Sheets appends), synchronized with live local n8n runtime, safely inactive.

---

### 4. VenturelyHub Outbound Generation

- **Repository Filename:** `02_outbound/outbound_generation.json`
- **Workflow Name:** `VenturelyHub Outbound Generation`
- **Live n8n Workflow ID:** `vhOutboundGen001`
- **Active Status:** `INACTIVE` (`active: false`)
- **Total Nodes:** 16 nodes (14 functional execution nodes + 2 sticky documentation notes)
- **Purpose:** Multi-channel outbound copy synthesis and draft generation engine:
  1. Trigger: `POST /webhook/outbound-generation`
  2. Input Validation: Enforces required `prospect_profile` and `venturelyhub_analysis`, captures source execution ID, flags synthetic test fixtures
  3. Payload Normalization: Normalizes verified lists, prepares structured evidence context
  4. Outbound Prompt Preparation: Injects VenturelyHub service matrix, verified prospect facts, and strict anti-hallucination rules
  5. AI Outbound Generation (Ollama Qwen3 8B): Synthesizes personalization analysis, 4-touch email sequence (Email 1, Follow-up 1, Follow-up 2, Follow-up 3), 3-touch LinkedIn sequence, and compact meeting pitch brief
  6. Response Parsing & Fallback: Parses Ollama JSON mode output with fallback protection
  7. Claim Validation Layer: Scans generated text against Phase 1 evidence to guarantee zero unsupported funding/hiring/tech claims
  8. Human Review Safeguard: Enforces `approval_status: "PENDING_REVIEW"` on every generated package. Zero automatic sending (no Gmail/LinkedIn execution)
  9. Persistent Audit Append: Appends record to `VenturelyHub` Google Spreadsheet (`Outbound Generation Outputs` on success, `System Events` on validation failure) with non-blocking error handling (`continueRegularOutput`)
  10. Direct Response: Delivers complete structured outbound package directly to caller
- **Dependencies:**
  - Ingests structured JSON payload produced by Phase 1 (`vhProspectInt001`)
- **Credential Requirements & Verification Status:**
  - Local Ollama API: Zero-credential HTTP endpoint (`http://127.0.0.1:11434/api/chat`, model `qwen3:8b`) — `AUTHENTICATION_VERIFIED / NATIVE_LOCAL`
  - Google Sheets OAuth2 API (`googleSheetsOAuth2Api`): Credential ID `JQRGvtvkfjEPF9WK` (`srisaikirantambalkar@gmail.com`) — `AUTHENTICATION_VERIFIED`
  - Gmail / LinkedIn: Strictly omitted; no sending permitted in Phase 2
- **Target Resources:**
  - Webhook endpoint: `/webhook/outbound-generation`
  - Spreadsheet: `VenturelyHub` (ID: `15__ZAea7cXNS0U3sd-GzsTuZWSFMIgomM_EHPjUr5N4`)
  - Target Worksheets: `Outbound Generation Outputs`, `System Events`
- **Current Status:** Implemented, 100% node validation pass, live runtime verified (Executions 13-18 verified across all 6 test scenarios with Qwen3 8B local inference and live Google Sheets appends), synchronized with live local n8n runtime, safely inactive.

---

### 5. VenturelyHub Outreach Approval & Execution

- **Repository Filename:** `03_outreach_execution/outreach_execution.json`
- **Workflow Name:** `VenturelyHub Outreach Approval & Execution`
- **Live n8n Workflow ID:** `vhOutboundExec001`
- **Active Status:** `INACTIVE` (`active: false`)
- **Total Nodes:** 15 nodes (13 functional execution nodes + 2 sticky documentation notes)
- **Purpose:** Gated outbound dispatch engine enforcing human authorization and executing verified email dispatches:
  1. Trigger: `POST /webhook/outreach-execution`
  2. Input Validation & Integrity Guard: Enforces `approval_status == "APPROVED"`, `claims_validation_status == "VERIFIED"`, valid email syntax, test mode recipient restrictions (`srisaikirantambalkar@gmail.com`), and verifies deterministic content hash (`approved_message_hash == h_<fnv1a64>`).
  3. Replay Protection: Evaluates `send_key` (`md5(recipient_email + "::" + subject + "::" + source_execution_id)`) to prevent duplicate outreach.
  4. Gmail Dispatch: Sends exact approved email text via Gmail integration node with `onError: continueRegularOutput` for fault isolation.
  5. Audit Formulation: Compiles execution metadata (`execution_id`, `message_id`, `send_status`, `sent_at`).
  6. Persistent Audit Append: Appends record to `VenturelyHub` Google Spreadsheet (`Outreach Execution Outputs` tab on dispatch attempt, `System Events` tab on rejection or duplicate prevent).
  7. Direct Caller Response: Returns complete execution outcome JSON directly to caller.
- **Dependencies:**
  - Ingests approved draft packages produced by Phase 2 (`vhOutboundGen001`) with explicit human authorization.
- **Credential Requirements & Verification Status:**
  - Google Sheets OAuth2 API (`googleSheetsOAuth2Api`): Credential ID `JQRGvtvkfjEPF9WK` (`srisaikirantambalkar@gmail.com`) — `AUTHENTICATION_VERIFIED`
  - Gmail OAuth2 API (`gmailOAuth2`): Credential ID `Z5LunU63lEhN8WRL` (`srisaikirantambalkar@gmail.com`) — `AUTHENTICATION_VERIFIED`
- **Target Resources:**
  - Webhook endpoint: `/webhook/outreach-execution`
  - Spreadsheet: `VenturelyHub` (ID: `15__ZAea7cXNS0U3sd-GzsTuZWSFMIgomM_EHPjUr5N4`)
  - Target Worksheets: `Outreach Execution Outputs`, `System Events`
- **Current Status:** Implemented, 100% node validation pass, live runtime verified in Phase 3.1 (Executions 32, 41 controlled sends with real Gmail provider message IDs; Executions 36, 42 replay blocks; Executions 37, 38, 39, 40 regression tests), synchronized with live local n8n runtime, safely inactive.

---

### 6. VenturelyHub Reply Intelligence (Phase 4 — Design & Architecture)

- **Repository Path:** `04_reply_intelligence/` (Architecture & Contract Design)
- **Workflow Name:** `VenturelyHub Reply Intelligence`
- **Planned n8n Workflow ID:** `vhReplyInt001`
- **Planned Status:** `INACTIVE` (`active: false`)
- **Purpose:** Inbound email response evaluation and commercial reasoning pipeline:
  1. Trigger Layer: Primary production ingestion via filtered native Gmail Trigger (`n8n-nodes-base.gmailTrigger`) targeting unread replies, with support for webhook-pinned test fixtures (`POST /webhook/reply-intelligence`).
  2. Deduplication Layer: Two-tier deduplication hierarchy combining runtime static data suppression with cross-restart Google Sheets audit lookup on `message_id`. Exactly-once processing is not claimed; idempotent suppression is strictly enforced.
  3. Correlation Layer: Resolves inbound replies to previous Phase 3 outbound executions via In-Reply-To, References, and Gmail thread IDs.
  4. Message Normalization: Strips quotation history, removes legal disclaimers, and normalizes sender metadata.
  5. AI Reasoning (Local Ollama Qwen3 8B): Classifies message into a 16-category taxonomy, extracts explicit vs inferred buying signals and objections, and drafts a strategic response brief.
  6. Deterministic Safety Enforcement: Hardcodes `send_allowed: false`. Evaluates mandatory human review policies (pricing, negotiations, complaints, legal, low confidence, and unsubscribe detection). Keeps complaint classification separate from opt-out intent. Recommends `MARK_FOR_MANUAL_SUPPRESSION` without maintaining a suppression database.
  7. Audit Storage: Appends record to `VenturelyHub` Google Spreadsheet (`Reply Intelligence Outputs` on evaluation, `System Events` on operational error).
  8. Direct Caller Response: Delivers complete structured intelligence JSON payload directly to caller.
- **Dependencies:**
  - Correlates with Phase 3 outbound dispatch records in `Outreach Execution Outputs`.
  - Dispatches follow-up responses strictly through Phase 3 (`vhOutboundExec001`) after explicit human approval.
- **Credential Requirements:**
  - Google Sheets OAuth2 API (`googleSheetsOAuth2Api`): Credential ID `JQRGvtvkfjEPF9WK` (`srisaikirantambalkar@gmail.com`)
  - Gmail OAuth2 API (`gmailOAuth2`): Credential ID `Z5LunU63lEhN8WRL` (`srisaikirantambalkar@gmail.com`) for inbound monitoring and thread fetching
  - Local Ollama API: Zero-credential HTTP endpoint (`http://127.0.0.1:11434/api/chat`, model `qwen3:8b`)
- **Target Resources:**
  - Ingestion entry point: `n8n-nodes-base.gmailTrigger` (primary production) / `/webhook/reply-intelligence` (testing)
  - Spreadsheet: `VenturelyHub` (ID: `15__ZAea7cXNS0U3sd-GzsTuZWSFMIgomM_EHPjUr5N4`)
  - Target Worksheets: `Reply Intelligence Outputs`, `System Events`
- **Current Status:** Architecture & Contract Design Hardening (Phase 4.0.1) complete. Detailed contracts, single primary ingestion entry point, complaint/suppression decoupling, and 24-scenario test matrix documented in `04_reply_intelligence/README.md`. Awaiting implementation authorization.
