# VenturelyHub Marketing & Sales Automation
## System Architecture Blueprint

This document defines the high-level technical architecture, operational boundaries, and component interactions of the **VenturelyHub Marketing & Sales Automation** system.

---

### 1. Architectural Guardrails & CRM Boundary

```
+-------------------------------------------------------------+
|               VENTURELYHUB CRM (OUT OF SCOPE)               |
|  - Completely external to this automation system            |
|  - No database synchronization, no pipeline replication      |
+-------------------------------------------------------------+
                              |
                     [Strict Air Gap]
                              |
+-------------------------------------------------------------+
|          VENTURELYHUB MARKETING & SALES AUTOMATION          |
|                                                             |
|  - Stateless workflow orchestration via n8n                 |
|  - Dual-LLM intelligence engine (Gemini + Local Ollama Qwen3 8B) |
|  - Explicit human control on all outbound actions           |
+-------------------------------------------------------------+
```

#### The Absolute CRM Rule
1. This project is **not** a CRM and will never become a CRM.
2. No deal-management, lead-management, customer, or prospect databases exist within this architecture.
3. Workflows operate as pure transformation and analytical pipelines, returning structured intelligence to downstream tasks or human reviewers.

---

### 2. Dual-AI Model Division of Responsibilities

The system leverages specialized models to enforce a clear separation between **objective facts** and **strategic analysis**:

```
                       [Target Prospect Input]
                                  │
                                  ▼
      ┌────────────────────────────────────────────────────────┐
      │             GOOGLE GEMINI (RESEARCH ENGINE)            │
      │  - Public web search & domain exploration              │
      │  - Factual data collection (programs, tech stacks)     │
      │  - Mandatory source URL attribution                    │
      │  - Grounded extraction into structured JSON schema     │
      │  - Zero extrapolation or hallucination allowed         │
      └────────────────────────────────────────────────────────┘
                                  │
                       [Verified Factual Data]
                                  │
                                  ▼
      ┌────────────────────────────────────────────────────────┐
      │         LOCAL OLLAMA / QWEN3 8B (STRATEGY ENGINE)      │
      │  - VenturelyHub service mapping & relevance            │
      │  - Qualitative fit classification (HIGH/MED/LOW/INSUFF)│
      │  - Tailored offer design & value proposition           │
      │  - Personalization angles & objection anticipation     │
      │  - Strict rule: Claude does NOT invent new facts       │
      └────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                 [Structured Intelligence Output]
```

---

### 3. Component Pipeline Architecture

#### Phase 1: Prospect Intelligence Pipeline (`01_prospect_intelligence/`)
- **Trigger Layer:** Webhook receiving target organization metadata (`POST /webhook/prospect-intelligence`).
- **Validation Layer:** Enforces input hygiene, required organization name, and test fixture detection.
- **Research Layer:** Gemini-powered search with built-in tools (`googleSearch`, `urlContext`).
- **Structuring Layer:** Normalizes facts, verifies data density, triggers defensive fallbacks if information is sparse.
- **Analysis Layer:** Local Ollama / Qwen3 8B powered commercial fit analysis mapping against VenturelyHub core service domains.
- **Output Formatting Layer:** Constructs standard contract-compliant JSON payload.
- **Audit Storage Layer:** Validates sheet payload and appends execution records to `VenturelyHub` spreadsheet (`Prospect Intelligence Outputs` on success, `System Events` on validation failure) with non-blocking error handling (`continueRegularOutput`).
- **Direct Response Layer:** Returns complete structured intelligence payload directly to the caller.

#### Phase 2: Personalization & Outbound Generation Pipeline (`02_outbound/`)
- **Trigger Layer:** Webhook receiving Phase 1 prospect intelligence payload (`POST /webhook/outbound-generation`).
- **Validation Layer:** Enforces required sections (`prospect_profile`, `venturelyhub_analysis`), captures source execution ID, flags synthetic test fixtures.
- **Normalization Layer:** Cleans arrays, prepares structured evidence context.
- **Prompting & Grounding Layer:** Injects VenturelyHub service matrix, verified prospect facts, and strict anti-hallucination rules.
- **Synthesis Layer:** Local Ollama / Qwen3 8B generates 4-touch email sequence, 3-touch LinkedIn sequence, compact meeting pitch brief, and personalization analysis.
- **Claim Validation Layer:** JavaScript filter scans generated copy against Phase 1 evidence to guarantee zero unsupported claims.
- **Human Review Safeguard:** Enforces `approval_status: "PENDING_REVIEW"` on every generated package. Zero automatic sending.
- **Audit Storage Layer:** Appends execution record to `VenturelyHub` spreadsheet (`Outbound Generation Outputs` on success, `System Events` on error).
- **Direct Response Layer:** Delivers complete multi-channel campaign package directly to caller.

#### Phase 3: Outreach Approval & Execution Pipeline (`03_outreach_execution/`)
- **Trigger Layer:** Webhook receiving approved draft payload (`POST /webhook/outreach-execution`).
- **Validation & Tamper Detection Layer:** Enforces `approval_status == "APPROVED"`, `claims_validation_status == "VERIFIED"`, valid email syntax, test recipient restrictions (`srisaikirantambalkar@gmail.com`), and validates deterministic content hash (`approved_message_hash == h_<fnv1a64>`).
- **Replay Protection Layer:** Evaluates `send_key` (`md5(recipient_email + "::" + subject + "::" + source_execution_id)`) to prevent duplicate outreach.
- **Dispatch Layer:** Gmail node sends approved subject and body verbatim. Zero LLM rewriting at send time. Configured with `onError: continueRegularOutput` for resilient failure isolation.
- **Audit Formulation Layer:** Formulates execution record including `execution_id`, `message_id`, and `send_status`.
- **Audit Storage Layer:** Appends execution record to `VenturelyHub` spreadsheet (`Outreach Execution Outputs` tab on dispatch attempt, `System Events` tab on rejection or duplicate prevent).
- **Direct Response Layer:** Delivers complete execution status directly to caller.

#### Phase 4: Reply Intelligence Pipeline (`04_reply_intelligence/`)
- **Trigger Layer:** Primary production ingestion via filtered native Gmail Trigger (`n8n-nodes-base.gmailTrigger`) targeting unread replies, with support for webhook-pinned test fixtures (`POST /webhook/reply-intelligence`).
- **Deduplication Layer:** Two-tier deduplication hierarchy combining runtime static data suppression with cross-restart Google Sheets audit lookup on `message_id`. Exactly-once processing is not claimed; idempotent suppression is strictly enforced.
- **Correlation Layer:** Resolves inbound replies to previous Phase 3 outbound executions via In-Reply-To, References, and thread IDs.
- **Normalization Layer:** Cleans quotation history, strips disclaimer boilerplate, and normalizes sender metadata.
- **Intelligence & Strategy Layer:** Local Ollama Qwen3 8B (`qwen3:8b`) interprets commercial intent, classifies response into 16-category taxonomy, identifies buying signals/objections, and drafts a suggested response strategy.
- **Deterministic Safety Guard:** Enforces `send_allowed: false` across all paths. Evaluates mandatory human review policies (pricing, negotiations, complaints, legal, low confidence, and unsubscribe detection). Keeps complaint classification separate from opt-out intent. Recommends `MARK_FOR_MANUAL_SUPPRESSION` without maintaining a suppression database.
- **Audit Storage Layer:** Appends execution record to `VenturelyHub` spreadsheet (`Reply Intelligence Outputs` on evaluation, `System Events` on operational error).
- **Direct Response Layer:** Returns complete structured intelligence JSON payload directly to caller.

---

### 4. Cross-Phase Google Sheets Execution & Output Audit Store

To provide persistent auditability, human inspection, and cross-phase pipeline coordination without violating the Absolute CRM Rule, all marketing automation phases append their execution outputs to a single centralized Google Spreadsheet:

**Target Spreadsheet:** `VenturelyHub`

#### Planned Target Worksheets:
1. **`Prospect Intelligence Outputs`** — Structured outputs from Phase 1 (factual profile, strategic fit, recommended offers, raw JSON).
2. **`Outbound Generation Outputs`** — Draft message variants, personalization angles, and approved messaging from Phase 2.
3. **`Outreach Execution Outputs`** — Dispatch logs, send timestamps, message IDs, and channel routing from Phase 3.
4. **`Reply Intelligence Outputs`** — Inbound sentiment analysis, intent categorization, and suggested replies from Phase 4.
5. **`Meeting & Sales Assistance Outputs`** — Pre-call briefs, attendee dossiers, and discussion frameworks from Phase 5.
6. **`Content Intelligence Outputs`** — Marketing content drafts, topic research, and social snippets from Phase 6.
7. **`Performance Intelligence Outputs`** — Aggregated weekly/monthly campaign analytics and conversion telemetry from Phase 7.
8. **`System Events`** — Cross-phase operational event log capturing validation failures, fallback invocations, and error events.

#### Core Architectural Sequence Rule:
Every phase workflow must follow the strict execution sequence:
```
Input
  ↓
Processing / Research / AI Analysis
  ↓
Structured Output Construction
  ↓
Output & Sheet Payload Validation
  ↓
Google Sheets Append (`VenturelyHub`)
  ↓
Final Structured Response to Caller
```

#### Key Operating Principles:
1. **Direct Response Integrity:** Google Sheets operates as a persistent audit/output layer while the workflow continues to return the structured response to the caller. The webhook caller always receives the complete structured JSON response directly.
2. **Synchronous Execution Model:** The Google Sheets append node executes within the active n8n execution pipeline between schema validation and final output response, providing verified persistence before response delivery.
3. **Traceability & Idempotency Boundary:** `execution_id` provides unique execution-level traceability. The current append-only audit layer does not guarantee duplicate suppression across manual/replayed executions. No CRM-style entity deduplication is performed.
4. **Full Raw JSON Preservation:** Every row appended to Google Sheets includes `raw_output_json` (or `raw_event_json` in `System Events`), ensuring zero data loss and enabling complete reconstructibility.
5. **Resilient Non-Blocking Execution:** Google Sheets append nodes are configured with `continueRegularOutput`. If credentials or external APIs encounter transient network issues, the execution completes gracefully and returns output to the caller.
6. **Absolute CRM Rule Compliance:** The `VenturelyHub` spreadsheet is strictly an execution, output, and audit sink. It does NOT function as a customer database, lead database, deal pipeline, sales pipeline, or CRM.
7. **Authentication & Runtime Verification:** Authenticated via n8n's native Google Sheets OAuth2 (`googleSheetsOAuth2Api`) using `srisaikirantambalkar@gmail.com`. Connected to verified spreadsheet `VenturelyHub` (ID: `15__ZAea7cXNS0U3sd-GzsTuZWSFMIgomM_EHPjUr5N4`).

---

### 5. Safety & Human Governance Protocol

1. **No Autonomous Outbound:** Automation creates structured intelligence, drafts, and recommendations. It never sends unreviewed external emails or commits commercial terms autonomously.
2. **Execution State:** All production workflows remain **INACTIVE** in the n8n runtime until explicitly authorized.
3. **Audit Trail:** Every workflow version and schema modification is version-controlled in Git and reconciled with the live local n8n runtime.


