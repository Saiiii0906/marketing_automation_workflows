# VenturelyHub Reply Intelligence System (Phase 4)
## Architecture & Contract Specification

---

## 1. Executive Summary & Purpose

The **VenturelyHub Reply Intelligence System** is the inbound response evaluation and commercial reasoning engine of the VenturelyHub Marketing & Sales Automation system.

Its sole purpose is to detect, normalize, interpret, and commercially classify inbound email responses received in response to VenturelyHub outreach, providing structured decision support for SriSaiKiran and the VenturelyHub team.

### Absolute Scope Boundaries & Architectural Guardrails
1. **Intelligence Only — Zero Autonomous Sending:**
   - Phase 4 is strictly an analytical and interpretative pipeline.
   - The contract output explicitly mandates `safety.send_allowed: false`.
   - The workflow **never** sends an automated email, creates an automated draft in Gmail, or triggers external communications.
   - Any follow-up or reply that is eventually approved by a human operator must flow through **Phase 3: VenturelyHub Outreach Approval & Execution** as the sole gated dispatch path.
2. **Absolute CRM Rule Compliance:**
   - This project is **not** a CRM and will never become a CRM.
   - Phase 4 contains **no** CRM connections, **no** customer/contact databases, **no** deal records, **no** sales pipelines, and **no** lifecycle state machines.
   - Persistent storage is restricted exclusively to the `VenturelyHub` Google Spreadsheet (`Reply Intelligence Outputs` on successful evaluation, `System Events` on operational error).
3. **No Send-Time Hallucination or Extrapolation:**
   - Commercial analysis clearly distinguishes between **EXPLICIT** facts stated by the prospect and **INFERRED** strategic signals.
   - Factual assertions cannot be invented.
4. **Deterministic Safety Over AI:**
   - Safety rules (unsubscribe detection, complaint escalation, mandatory human review for pricing/legal/commitments) are deterministically enforced in code and cannot be overridden by LLM inferences.

---

## 2. Inbound Gmail Ingestion Architecture

### Primary Production Ingestion Mechanism: Native Gmail Trigger (`n8n-nodes-base.gmailTrigger`)

Based on our inspection of the live n8n installation (`v2.41.3`), the workflow uses **`n8n-nodes-base.gmailTrigger` (v1.4)** as its **single primary production entry point**.

We selected `n8n-nodes-base.gmailTrigger` because:
1. It is the native, battle-tested n8n polling trigger designed specifically for event-driven Gmail message fetching.
2. It directly leverages the authorized `gmailOAuth2` credential (`Z5LunU63lEhN8WRL` - `srisaikirantambalkar@gmail.com`).
3. It avoids unnecessary architectural bifurcation by serving as the unified entry point for inbound reply monitoring.

### Ingestion Filter Specifications
To strictly avoid processing arbitrary, personal, or promotional mail, the `gmailTrigger` is configured with deterministic query filters:
- **Search Query (`filters.q`):**
  ```
  to:srisaikirantambalkar@gmail.com -from:srisaikirantambalkar@gmail.com is:unread -category:promotions -category:spam
  ```
- **Read Status (`filters.readStatus`):** `unread`
- **Drafts Excluded (`filters.includeDrafts`):** `false`
- **Spam & Trash Excluded (`filters.includeSpamTrash`):** `false`
- **Polling Interval (`pollTimes`):** Configurable interval (default: every 5 minutes in production; manually triggered or pinned in testing).
- **Lightweight Fetching (`simple: false` downstream):** Trigger fetches lightweight metadata (`simple: true`), and full MIME headers (`In-Reply-To`, `References`, body) are retrieved as needed to conserve memory and avoid OOM risks.

---

## 3. Conversation & Thread Correlation Strategy

To determine which Phase 3 outbound execution an incoming reply corresponds to, the system uses a 4-tier hierarchy:

```
[Incoming Inbound Message]
            │
    [Tier 1: In-Reply-To / References Header Matching]
            │ Match? ──► YES ──► Status: CORRELATED (High Confidence)
            ▼ NO
    [Tier 2: Gmail threadId Direct Match against Outreach Execution Outputs]
            │ Match? ──► YES ──► Status: CORRELATED (High Confidence)
            ▼ NO
    [Tier 3: Source Execution ID Token in Subject / Body]
            │ Match? ──► YES ──► Status: CORRELATED (Medium Confidence)
            ▼ NO
    [Tier 4: Recipient Domain / Normalized Subject Heuristic]
            │ Match? ──► YES ──► Status: CORRELATED_HEURISTIC (Low Confidence)
            ▼ NO
    [Correlation Unresolved]
            └── Status: UNRESOLVED (source_execution_id: null)
```

### Correlation Rules:
1. **Never Invent Correlation:** If the email does not match an existing Phase 3 outbound execution ID or message ID in `Outreach Execution Outputs`, `correlation_status` must strictly equal `"UNRESOLVED"`.
2. **Subject-Line Matching is Non-Authoritative:** Matching solely on subject text (e.g., "Re: Hello") is never treated as definitive correlation without secondary sender email verification.
3. **Traceability:** When correlated, the system extracts:
   - `source_execution_id` (e.g., `EXEC-P3-20261001-3236`)
   - `source_outbound_subject`
   - `original_campaign_angle`
   - `prospect_organization_name`

---

## 4. Execution-Level Deduplication Strategy

To prevent repeatedly analyzing the same inbound message across polling intervals:

1. **Composite Deduplication Identity:**
   $$\text{dedup\_key} = \text{message\_id} \parallel \text{"::"} \parallel \text{thread\_id} \parallel \text{"::"} \parallel \text{sender\_email}$$
2. **Two-Tier Deduplication Hierarchy:**
   - **Tier 1 (Runtime-Level):** Evaluated in `CHECK_AlreadyProcessed` via n8n workflow static data (`$getWorkflowStaticData('global').processedMessageIds`). This provides fast in-memory suppression during continuous active runtime execution.
   - **Tier 2 (Cross-Restart Persistence):** When workflow runtime restarts or resets static data, the deduplication node checks the `Reply Intelligence Outputs` worksheet in Google Sheets (or checks `System Events`) for previously processed `source_message_id` records before proceeding.
3. **Execution Gate Routing:**
   - If `dedup_key` or `source_message_id` already exists:
     - The workflow immediately branches to `DUPLICATE_PREVENTED`.
     - Logs the event to `System Events` in Google Sheets.
     - Returns structured status: `status: "ALREADY_PROCESSED"`.
     - LLM inference and downstream processing are strictly bypassed.
4. **Guarantees & Exact Boundaries:**
   - **At-least-once ingestion** with idempotent suppression is guaranteed.
   - **"Exactly-once" processing is NOT claimed**, because distributed network polling and external API retries can theoretically deliver duplicate webhook/poll triggers before state is committed. The architecture enforces idempotent handling rather than theoretical exactly-once delivery.

---

## 5. Reply Classification Taxonomy

The system defines 16 mutually exclusive primary classifications:

| Category | Definition | Default Human Review |
| :--- | :--- | :---: |
| `POSITIVE_INTEREST` | Prospect expresses general enthusiasm, alignment, or interest in VenturelyHub capabilities | Recommended |
| `MEETING_REQUEST` | Prospect explicitly proposes or asks for a demo, call, or discussion | **Mandatory** |
| `INFORMATION_REQUEST` | Prospect asks technical, portfolio, case study, or process questions | Optional |
| `PRICING_REQUEST` | Prospect asks for rate cards, project estimates, or pricing models | **Mandatory** |
| `NEGOTIATION` | Prospect discusses commercial terms, rate reductions, or contractual conditions | **Mandatory** |
| `OBJECTION` | Prospect raises specific hesitations (e.g., already have agency, budget frozen, tech mismatch) | Recommended |
| `NOT_INTERESTED` | Prospect politely or firmly declines services without hostility | Optional |
| `UNSUBSCRIBE` | Prospect requests removal, stops future emails, or mentions spam | **Mandatory** |
| `OUT_OF_OFFICE` | Automated vacation, leave, or sabbatical autoresponder | Automatically Classified |
| `AUTOMATED_REPLY` | Delivery receipts, generic ticketing confirmation, or system auto-responses | Automatically Classified |
| `COMPLAINT` | Prospect expresses anger, frustration, or negative feedback regarding contact | **Mandatory** |
| `AMBIGUOUS` | Unclear, cryptic, single-word, or context-free response | **Mandatory** |
| `WRONG_RECIPIENT` | Prospect indicates they are not the appropriate department, contact, or company | Optional |
| `REFERRAL` | Prospect directs outreach to another colleague or executive within the company | Recommended |
| `FOLLOW_UP` | Prospect asks to reconnect at a specific future date or quarter | Recommended |
| `OTHER` | Unclassified message not matching existing taxonomy definitions | **Mandatory** |

---

## 6. Commercial Intelligence Contract

For all meaningful prospect replies (`POSITIVE_INTEREST`, `MEETING_REQUEST`, `INFORMATION_REQUEST`, `PRICING_REQUEST`, `NEGOTIATION`, `OBJECTION`, `REFERRAL`, `FOLLOW_UP`), the engine extracts:

```json
{
  "commercial_signals": {
    "intent": "Explicit prospect intent summary",
    "interest_level": "HIGH | MEDIUM | LOW | NONE",
    "urgency": "IMMEDIATE | HIGH | NORMAL | LOW | DEFERRED",
    "sentiment": "POSITIVE | NEUTRAL | SKEPTICAL | NEGATIVE | HOSTILE",
    "buying_signals": [
      {
        "signal": "Active need to build AI MVP next month",
        "type": "EXPLICIT",
        "confidence": 1.0
      }
    ],
    "objections": [
      {
        "objection": "Existing in-house team handles core frontend",
        "type": "EXPLICIT",
        "category": "INTERNAL_CAPABILITY"
      }
    ],
    "questions": [
      "Can VenturelyHub handle mobile Flutter development as well as Web?"
    ],
    "requirements": [
      "Must have SOC2 compliance or HIPAA experience"
    ],
    "timeline": {
      "stated_timeframe": "Next month (Q4 launch)",
      "is_explicit": true
    },
    "decision_maker_signals": {
      "role_indicated": "CTO / Technical Decision Maker",
      "authority_level": "FINAL_DECISION_MAKER | INFLUENCER | GATEKEEPER | UNKNOWN"
    }
  }
}
```

---

## 7. Response Strategy & Recommendation Contract

Ollama (`qwen3:8b`) produces a structured response plan:

```json
{
  "response_strategy": {
    "summary": "Prospect is eager to build an AI MVP in November and is asking for engineering availability and blended rates.",
    "key_points": [
      "Startup has immediate Q4 runway and needs AI engineering",
      "Needs rate card and team lead availability"
    ],
    "recommended_next_action": "SCHEDULE_CALL | PROVIDE_INFO | SEND_PRICING | ESCALATE_TO_FOUNDER | MARK_UNSUBSCRIBE | CLOSE_LOOP",
    "response_angle": "Validate technical feasibility for AI MVP, suggest 20-min architectural scoping call, provide high-level pricing ranges.",
    "important_topics_to_address": [
      "VenturelyHub LLM engineering velocity",
      "Senior engineer onboarding timeline"
    ],
    "suggested_cta": "Would Thursday at 3 PM or Friday morning work for a brief 20-minute architectural discussion?",
    "suggested_reply_draft": "Hi [Name],\n\nThanks for getting back to us. We have direct experience spinning up AI MVPs within 3-4 weeks...\n\nBest,\nSriSaiKiran",
    "draft_disclaimer": "INTELLIGENCE ARTIFACT ONLY. DISPATCH STRICTLY PROHIBITED WITHOUT PHASE 3 HUMAN APPROVAL."
  }
}
```

---

## 8. Deterministic Human Review & Safety Policies

### 1. Mandatory Human Review Matrix
`safety.human_review_required` is evaluated using deterministic code rules:
- **Pricing & Commercial Terms:** If `PRICING_REQUEST` or `NEGOTIATION` $\rightarrow$ `human_review_required: true`.
- **Legal & Commitments:** Any mention of SLAs, guarantees, contracts, or warranties $\rightarrow$ `human_review_required: true`.
- **Complaints & Grievances:** If `COMPLAINT` $\rightarrow$ `human_review_required: true`.
  - `do_not_contact = true` **ONLY** when explicit opt-out or do-not-contact language is detected in the message.
  - Otherwise `do_not_contact = false`.
  - `unsubscribe_detected = true` **ONLY** when explicit unsubscribe / opt-out intent exists.
  - Keeps complaint classification strictly separate from unsubscribe intent.
- **Unsubscribe Requests:** If `UNSUBSCRIBE` $\rightarrow$ `human_review_required: true`, `unsubscribe_detected: true`, `do_not_contact: true`.
- **Low Confidence:** If classification confidence $< 0.80$ $\rightarrow$ `human_review_required: true`, `review_reason: ["LOW_CONFIDENCE_CLASSIFICATION"]`.

### 2. Unsubscribe & Suppression Recommendation Policy
- **No Suppression Database:** Phase 4 does **NOT** create, maintain, or modify any suppression database, CRM contact list, or campaign database.
- When `unsubscribe_detected: true`:
  - `send_allowed: false`
  - `do_not_contact: true`
  - `recommended_next_action: "MARK_FOR_MANUAL_SUPPRESSION"` (intelligence recommendation only for human review)
  - Logged to `System Events` for human operator inspection.
  - Zero auto-replies generated.

### 3. Out-Of-Office (OOO) Policy
- Out-of-office autoresponders are **automatically classified** (`reply_type: "OUT_OF_OFFICE"`).
- Extracts explicitly stated `return_date` if present in the text.
- Extracts explicitly stated `alternate_contact` if present in the text.
- Recommends next action (e.g., `DEFER_FOLLOW_UP_UNTIL_RETURN`).
- **Never** automatically sends a response or creates automated follow-up calendar tasks.

---

## 9. Local Ollama Integration Contract

- **Endpoint:** `http://127.0.0.1:11434/api/chat`
- **Model:** `qwen3:8b`
- **Format:** `format: "json"`
- **Options:** `temperature: 0.1` (low temperature for deterministic classification)
- **Validation Pipeline:**
  1. POST prompt to Ollama with strict JSON schema definition.
  2. Parse returned JSON string.
  3. Validate required fields (`reply_type`, `intent`, `interest_level`, `sentiment`, `summary`).
  4. Normalize enums against the taxonomy; fallback to `AMBIGUOUS` if invalid.
  5. Apply deterministic safety overrides.

---

## 10. Google Sheets Audit Schema

Worksheet: **`Reply Intelligence Outputs`** in Spreadsheet `VenturelyHub` (`15__ZAea7cXNS0U3sd-GzsTuZWSFMIgomM_EHPjUr5N4`).

| Column Header | Data Type | Description |
| :--- | :--- | :--- |
| `execution_id` | String | Unique execution identifier (e.g., `EXEC-P4-20261003-8821`) |
| `created_at` | ISO Timestamp | Execution timestamp |
| `workflow_name` | String | Fixed: `VenturelyHub Reply Intelligence` |
| `phase` | String | Fixed: `Phase 4 - Reply Intelligence` |
| `status` | String | `SUCCESS` or `FAILED` |
| `test_fixture` | Boolean | True for synthetic fixtures or test runs |
| `source_message_id` | String | Gmail provider message ID of the inbound reply |
| `source_thread_id` | String | Gmail thread ID |
| `source_execution_id` | String | Correlated Phase 3 outbound execution ID or `UNRESOLVED` |
| `correlation_status` | String | `CORRELATED`, `UNRESOLVED`, or `NOT_APPLICABLE` |
| `sender_identifier` | String | Sender email address |
| `reply_type` | String | Classified category from taxonomy |
| `intent` | String | Concise summary of prospect's intent |
| `interest_level` | String | `HIGH`, `MEDIUM`, `LOW`, or `NONE` |
| `urgency` | String | `IMMEDIATE`, `HIGH`, `NORMAL`, `LOW`, or `DEFERRED` |
| `sentiment` | String | `POSITIVE`, `NEUTRAL`, `SKEPTICAL`, `NEGATIVE`, or `HOSTILE` |
| `summary` | String | Executive summary of message content |
| `key_points` | String (JSON Array) | Key takeaways extracted from message |
| `questions` | String (JSON Array) | Explicit questions asked by prospect |
| `objections` | String (JSON Array) | Hesitations or blockers raised |
| `buying_signals` | String (JSON Array) | Specific indicators of purchase intent |
| `requirements` | String (JSON Array) | Stated technical or project criteria |
| `timeline` | String | Stated project timeframe or null |
| `recommended_next_action` | String | Action code (e.g., `SCHEDULE_CALL`, `PROVIDE_INFO`) |
| `recommended_response_strategy`| String | Strategic advice for the response |
| `suggested_reply` | String | Draft reply copy (intelligence artifact only) |
| `human_review_required` | Boolean | True if human review is mandatory |
| `review_reason` | String (JSON Array) | Array of policy trigger reasons |
| `unsubscribe_detected` | Boolean | True if opt-out requested |
| `do_not_contact` | Boolean | True if prospect must not be contacted |
| `send_allowed` | Boolean | Always strictly `FALSE` |
| `raw_output_json` | JSON String | Complete serialized execution payload |
| `error_message` | String | Error description if status is FAILED |

*Prohibited Columns:* `customer_id`, `account_id`, `deal_id`, `pipeline_stage`, `lead_status`, `sales_stage`, `CRM_record_id`.

---

## 11. Complete JSON Output Contract

```json
{
  "execution_id": "EXEC-P4-20261003-8821",
  "created_at": "2026-10-03T10:30:00.000Z",
  "workflow_name": "VenturelyHub Reply Intelligence",
  "phase": "Phase 4 - Reply Intelligence",
  "status": "SUCCESS",
  "test_fixture": false,
  "source": {
    "message_id": "1a0f8eef3102d853",
    "thread_id": "1a0f8eef3102d853",
    "source_execution_id": "EXEC-P3-20261001-3236",
    "correlation_status": "CORRELATED"
  },
  "sender": {
    "email": "client@startup.com",
    "name": "Jane Doe",
    "organization": "Acme Ventures"
  },
  "reply": {
    "reply_type": "MEETING_REQUEST",
    "intent": "Request discovery call to discuss AI engineering support",
    "interest_level": "HIGH",
    "urgency": "HIGH",
    "sentiment": "POSITIVE",
    "summary": "Prospect is interested in building an AI MVP next month and requested a call this week to review pricing and team availability.",
    "key_points": [
      "Startup has active Q4 budget for AI MVP",
      "Needs 2-3 engineers starting next month",
      "Requested a 20-min call Thursday"
    ],
    "questions": [
      "What are your typical hourly or monthly blended rates?",
      "Do you have senior Flutter developers available immediately?"
    ],
    "objections": [],
    "buying_signals": [
      "Stated urgent timeline ('next month')",
      "Direct request for pricing models"
    ],
    "requirements": [
      "AI / LLM engineering expertise",
      "Flutter mobile capability"
    ],
    "timeline": "Next month (November 2026)"
  },
  "commercial_intelligence": {
    "recommended_next_action": "SCHEDULE_CALL",
    "recommended_response_strategy": "Acknowledge AI MVP expertise, provide high-level pricing ranges, and confirm availability for Thursday at 3 PM.",
    "suggested_cta": "Would Thursday at 3 PM or Friday morning work for a brief 20-minute discussion?",
    "suggested_reply": "Hi Jane,\n\nThanks for reaching out! We would love to support your AI MVP rollout...\n\nBest,\nSriSaiKiran"
  },
  "safety": {
    "human_review_required": true,
    "review_reason": [
      "PRICING_REQUEST_DETECTED",
      "MEETING_REQUEST_ACTION"
    ],
    "unsubscribe_detected": false,
    "do_not_contact": false,
    "send_allowed": false
  },
  "metadata": {
    "model": "qwen3:8b",
    "latency_ms": 2450,
    "research_required": false
  }
}
```

---

## 12. Verification Test Matrix (22 Scenarios)

| # | Scenario Description | Input Inbound Message Characteristics | Expected Classification | Human Review | Expected Safety & Persistence Behavior |
| :---: | :--- | :--- | :--- | :---: | :--- |
| **1** | Positive Interest | "Sounds very relevant. We'd like to learn more about your AI work." | `POSITIVE_INTEREST` | Recommended | Appended to Sheets; `send_allowed: false` |
| **2** | Meeting Request | "Can we hop on a 15-minute call Thursday at 2 PM EST?" | `MEETING_REQUEST` | **Mandatory** | `recommended_next_action: SCHEDULE_CALL`; `send_allowed: false` |
| **3** | Pricing Request | "Could you send over your rate card and typical MVP project costs?" | `PRICING_REQUEST` | **Mandatory** | `review_reason: ["PRICING_REQUEST_DETECTED"]`; `send_allowed: false` |
| **4** | Negotiation | "Your rates are slightly high for our budget. Can you offer a 15% discount for a 6-month commitment?" | `NEGOTIATION` | **Mandatory** | `review_reason: ["NEGOTIATION_DETECTED"]`; `send_allowed: false` |
| **5** | Information Request | "Do your engineers have experience integrating LangChain and Pinecone?" | `INFORMATION_REQUEST` | Optional | `questions` extracted; `send_allowed: false` |
| **6** | Objection | "We already have an internal engineering team of 12 people." | `OBJECTION` | Recommended | `objections` extracted; `recommended_next_action: ADDRESS_OBJECTION` |
| **7** | Not Interested | "Thanks, but we do not need external tech partners at this time." | `NOT_INTERESTED` | Optional | `interest_level: NONE`; `recommended_next_action: CLOSE_LOOP` |
| **8** | Unsubscribe | "Please unsubscribe me and remove our company from your mailing list." | `UNSUBSCRIBE` | **Mandatory** | `unsubscribe_detected: true`, `do_not_contact: true`, `recommended_next_action: "MARK_FOR_MANUAL_SUPPRESSION"`, `send_allowed: false` |
| **9** | Out of Office | "I am out of the office until Oct 14 with limited access to email." | `OUT_OF_OFFICE` | Automatically Classified | `return_date: "2026-10-14"` extracted; no auto-reply; `send_allowed: false` |
| **10** | Automated System Reply | "Your message was received by Support Ticket #49201." | `AUTOMATED_REPLY` | Automatically Classified | Ignored from commercial pipeline; logged to System Events |
| **11** | Complaint (General) | "Your cold email interrupted my workday. This approach is annoying." | `COMPLAINT` | **Mandatory** | `human_review_required: true`, `do_not_contact: false` (no opt-out language), `unsubscribe_detected: false`, escalated in `System Events` |
| **12** | Complaint (With Opt-Out) | "Stop emailing our team immediately. Do not contact us again." | `COMPLAINT` | **Mandatory** | `human_review_required: true`, `do_not_contact: true`, `unsubscribe_detected: true`, `recommended_next_action: "MARK_FOR_MANUAL_SUPPRESSION"` |
| **13** | Ambiguous Reply | "Interesting." / "Hmm maybe." / "Ok." | `AMBIGUOUS` | **Mandatory** | `review_reason: ["AMBIGUOUS_REPLY"]`; `send_allowed: false` |
| **14** | Wrong Recipient | "I am in accounting, I don't handle technology decisions." | `WRONG_RECIPIENT` | Optional | `recommended_next_action: CLOSE_LOOP` |
| **15** | Referral | "I don't handle this, but please reach out to our VP of Eng, Sarah at sarah@acme.com." | `REFERRAL` | Recommended | Referred contact extracted; `human_review_required: true` |
| **16** | Multi-Question Complex | Reply with 4 detailed architectural questions and a compliance inquiry | `INFORMATION_REQUEST` | **Mandatory** | All 4 questions isolated into `questions` array |
| **17** | Correlated Gmail Thread | Message contains `In-Reply-To` matching Phase 3 message ID `1a0f8eef3102d853` | Any valid taxonomy | Based on type | `correlation_status: CORRELATED`; links to source execution |
| **18** | Unresolved Correlation | Inbound cold inquiry or reply with stripped headers and unknown subject | Any valid taxonomy | Based on type | `correlation_status: UNRESOLVED`; `source_execution_id: null` |
| **19** | Duplicate Inbound (Runtime) | Immediate resubmission of processed `message_id` within active runtime | N/A (Bypassed) | N/A | Intercepted by Tier 1 static data; logs `ALREADY_PROCESSED` to `System Events` |
| **20** | Duplicate Inbound (Cross-Restart) | Resubmission of processed `message_id` after workflow restart / cache clear | N/A (Bypassed) | N/A | Intercepted by Tier 2 Google Sheets audit lookup; logs `ALREADY_PROCESSED` to `System Events` |
| **21** | Malformed / Empty Body | Inbound email with 0 text content or null body | N/A | **Mandatory** | Validation fails; returns `MALFORMED_INPUT`; logs to `System Events` |
| **22** | Ollama Model Timeout | Local Ollama instance fails to respond within timeout window | N/A | **Mandatory** | Falls back to deterministic rule classifier; flags `FALLBACK_APPLIED` |
| **23** | Malformed Model JSON | Ollama outputs broken JSON syntax | N/A | **Mandatory** | JSON repair parser activated; falls back to rule classifier |
| **24** | Low-Confidence Inference | Model confidence $< 0.75$ on subtle or mixed message | `AMBIGUOUS` | **Mandatory** | Flags `LOW_CONFIDENCE_CLASSIFICATION`; human review required |

---

## 12. Downstream Phase 3 Integration Boundary

```
[Phase 4: Reply Intelligence]
         │
         ├── Interprets reply & extracts commercial intent
         ├── Generates suggested response strategy & draft
         ├── Strictly sets: send_allowed = false
         └── Logs record to: 'Reply Intelligence Outputs'
                    │
                    ▼
          [Human Reviewer in Google Sheets / UI]
                    │
                    ├── Reviews draft & commercial recommendations
                    ├── Authorizes response (approval_status: APPROVED)
                    └── Generates approved_message_hash
                                │
                                ▼
         [Phase 3: Outreach Approval & Execution]
                    │
                    ├── Validates cryptographic approval hash
                    ├── Enforces safe recipient & replay checks
                    └── Dispatches approved reply via Gmail OAuth2
```

---

## 13. Future Implementation Roadmap

When authorized for implementation:
1. **Module Creation:** Create `04_reply_intelligence/reply_intelligence.json` implementing the 14-node architecture.
2. **Sheet Header Migration:** Verify and initialize the complete 33-column schema in worksheet `Reply Intelligence Outputs`.
3. **Local Testing:** Execute the 22-scenario test suite in n8n manual runner.
4. **Synchronization:** Export live workflow definition, update documentation, and push commit to `origin/main`.
