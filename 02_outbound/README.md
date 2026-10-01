# VenturelyHub Outbound Generation Workflow (Phase 2)

## 1. Overview & Objective

The **VenturelyHub Outbound Generation** workflow (`02_outbound/outbound_generation.json`) is the multi-channel personalization and draft generation engine of the VenturelyHub Marketing & Sales Automation system.

Its sole purpose is to transform verified, grounded prospect intelligence (produced by Phase 1) into a complete, tailored outbound campaign package spanning multiple channels, while enforcing strict factual grounding, claim validation, and human review boundaries.

### Absolute CRM Rule Compliance & Audit Store
- **Execution & Output Store Only:** This workflow writes structured execution records to the unified `VenturelyHub` Google Spreadsheet (`Outbound Generation Outputs` tab on success, `System Events` tab on validation failure). It contains **no** CRM connections, **no** customer/deal databases, and does **not** manage a sales pipeline.
- **Persistent Output Architecture:** Google Sheets operates strictly as an append-only persistent execution/audit layer while returning complete structured draft packages directly to the HTTP caller.
- **Human Review Safeguard:** Every generated campaign package mandates `approval_status: "PENDING_REVIEW"`.
- **Zero Automated Dispatch:** This workflow generates drafts only. There is **no** automatic sending via Gmail, LinkedIn, or any external communications channel. Dispatch execution is reserved for Phase 3 after explicit human authorization.
- **Non-Blocking Resilience:** Google Sheets append nodes are configured with `onError: continueRegularOutput`. If transient issues occur, execution completes gracefully and returns output to the caller.

---

## 2. Technology Architecture & Division of Responsibility

| Component | Engine / Model | Operational Scope |
| :--- | :--- | :--- |
| **Orchestration** | n8n (`v2.41.3`) | Execution flow, schema validation, payload normalization, branching, error catching |
| **Input Source** | **Phase 1 Prospect Intelligence** (`vhProspectInt001`) | Verified factual prospect profile, research citations, and strategic fit signals |
| **Outbound Synthesis** | **Local Ollama / Qwen3 8B** (`qwen3:8b` via `http://127.0.0.1:11434/api/chat`) | Personalization analysis, 4-touch email sequence, 3-touch LinkedIn sequence, meeting pitch |
| **Claim Validation** | JavaScript Code (`VALIDATE_Claims`) | Anti-hallucination filter checking that factual claims match Phase 1 evidence |
| **Audit Storage** | **Google Sheets** (`VenturelyHub`) | Persistent output records (`Outbound Generation Outputs`) and error events (`System Events`) |

---

## 3. Workflow Architecture & Pipeline Sequence

```
PHASE 1 PROSPECT INTELLIGENCE PAYLOAD
     ↓
[TRIGGER_Phase1Input] (Webhook POST /webhook/outbound-generation)
     ↓
[VALIDATE_Input] (Validates required sections, flags synthetic fixtures, captures source execution ID)
     ↓
[ROUTE_Validation] (If: isValid === true)
     ├── FALSE → [FORMAT_ErrorOutput] 
     │                ↓
     │           [GOOGLE_SHEETS_Append_SystemEvent] (Appends error to 'System Events' tab)
     │                ↓
     │           [RETURN_ErrorOutput] (Returns structured error JSON to caller)
     ↓ TRUE
[NORMALIZE_Phase1Output] (Normalizes arrays, extracts verified fields, prepares clean context)
     ↓
[PREPARE_OutboundPrompt] (Injects VenturelyHub capability matrix, verified evidence, strict grounding rules)
     ↓
[GENERATE_Outbound_Ollama] (Calls local Ollama qwen3:8b for multi-channel copy generation)
     ↓
[PARSE_Outbound] (Parses Ollama JSON, activates deterministic fallback if needed)
     ↓
[VALIDATE_Claims] (Anti-hallucination scan ensuring zero unsupported funding/hiring/tech claims)
     ↓
[BUILD_OutputPayload] (Constructs contract-compliant final JSON, formats Google Sheets row)
     ↓
[GOOGLE_SHEETS_Append_Outbound] (Appends record to 'Outbound Generation Outputs' tab)
     ↓
[RETURN_Output] (Returns complete structured outbound package to caller)
```

---

## 4. Input Specification

The workflow accepts POST requests at `/webhook/outbound-generation` with a structured Phase 1 output payload:

### Primary Input Sections
```json
{
  "prospect_profile": {
    "organization_name": "iCreate",
    "website": "https://www.icreate.org.in",
    "type": "incubator",
    "location": "Ahmedabad, Gujarat, India",
    "description": "Autonomous centre of excellence facilitating tech innovation...",
    "focus_areas": ["DeepTech", "IoT", "GreenTech"],
    "industries": ["Cleantech", "Hardware/Embedded Systems"],
    "programs": ["Sparkup Idea Fund", "iCreate Incubation Program"],
    "technology_focus": ["AI/ML", "Embedded Systems"]
  },
  "research": {
    "relevant_activities": ["Annual startup demo day", "EV innovation challenge"],
    "partnership_signals": ["Partnership with tech accelerators"],
    "relevant_contact_roles": ["Relevant contact not publicly verified"],
    "sources": ["https://www.icreate.org.in"],
    "research_date": "2026-10-01"
  },
  "venturelyhub_analysis": {
    "fit_level": "HIGH",
    "fit_reasons": ["Direct alignment with startup engineering model."],
    "venturelyhub_relevance": "VenturelyHub can serve as a vetted engineering partner...",
    "relevant_services": ["Product Engineering", "AI Development", "Cloud Architecture"],
    "recommended_offer": "Technology Enablement Consultation",
    "value_proposition": "Provide scalable tech infrastructure...",
    "outreach_angle": "Supporting startup acceleration programs...",
    "recommended_contact_role": "Relevant contact not publicly verified",
    "recommended_cta": "Request a discovery call...",
    "confidence": "HIGH"
  },
  "metadata": {
    "execution_id": "EXEC-P1-20261001-9067",
    "test_fixture": true
  }
}
```

---

## 5. Output Specification

### Success Response Structure (`status: "SUCCESS"`)
```json
{
  "personalization_analysis": {
    "why_this_prospect": "Commercial rationale connecting prospect focus to VenturelyHub",
    "relevant_signal": "Verified activity or partnership signal from research",
    "personalization_angle": "Core thematic angle for personalization",
    "value_proposition": "Clear, compelling value proposition sentence",
    "recommended_offer": "Tailored VenturelyHub offer",
    "recommended_contact_role": "Target role title",
    "recommended_cta": "Low-friction call to action",
    "tone": "Professional, collaborative, evidence-based",
    "claims_to_avoid": ["Unsupported funding status", "Unverified tech stack details"],
    "risks": ["Context gaps identified"],
    "confidence": "HIGH | MEDIUM | LOW"
  },
  "email_sequence": {
    "email_1": { "subject": "...", "body": "...", "purpose": "Initial introduction and verified relevance" },
    "follow_up_1": { "subject": "...", "body": "...", "purpose": "Reinforce relevance and offer specific support" },
    "follow_up_2": { "subject": "...", "body": "...", "purpose": "Alternative angle or engineering capability" },
    "follow_up_3": { "subject": "...", "body": "...", "purpose": "Polite close-the-loop inquiry" }
  },
  "linkedin_sequence": {
    "connection_note": "Short natural note under 300 chars",
    "follow_up_1": { "message": "...", "purpose": "Reference verified context" },
    "follow_up_2": { "message": "...", "purpose": "Introduce VenturelyHub capability and soft CTA" }
  },
  "meeting_pitch": {
    "opening": "Why the conversation is relevant",
    "value_proposition": "What VenturelyHub can contribute",
    "suggested_discussion_points": ["Point 1", "Point 2", "Point 3"],
    "cta": "Reasonable next exploratory step"
  },
  "approval": {
    "approval_status": "PENDING_REVIEW",
    "review_notes": "All claims verified against Phase 1 evidence. Ready for human review."
  },
  "claims_validation": {
    "status": "VERIFIED",
    "violations_detected": [],
    "action_taken": "Passed strict claim grounding check"
  },
  "metadata": {
    "execution_id": "EXEC-P2-20261001-7306",
    "created_at": "2026-10-01T16:57:49.242Z",
    "workflow_name": "VenturelyHub Outbound Generation",
    "phase": "Phase 2 - Personalization & Outbound Generation",
    "test_fixture": true,
    "source_execution_id": "EXEC-P1-20261001-9067"
  }
}
```

---

## 6. Google Sheets Execution & Audit Schemas

The workflow appends records to the central `VenturelyHub` spreadsheet across two target worksheets:

### Worksheet: `Outbound Generation Outputs` (Success Path)
| Column Name | Type | Purpose / Description |
| :--- | :--- | :--- |
| `execution_id` | string | Unique execution identifier (e.g. `EXEC-P2-20261001-7306`) |
| `created_at` | string | ISO-8601 execution timestamp |
| `workflow_name` | string | Constant: `"VenturelyHub Outbound Generation"` |
| `phase` | string | Constant: `"Phase 2 - Outbound Generation"` |
| `status` | string | Execution outcome status: `"SUCCESS"` |
| `test_fixture` | boolean | `true` if synthetic input; `false` for genuine prospects |
| `organization_name`| string | Target organization name |
| `target_channel` | string | Target channel mix: `"Email + LinkedIn"` |
| `subject_line` | string | Generated primary email subject line |
| `message_body` | string | Generated primary email body |
| `personalization_angle` | string | Core thematic personalization angle |
| `raw_output_json` | string | Complete verbatim JSON output preserving all 4 emails, 3 LinkedIn messages, pitch, and metadata |

### Worksheet: `System Events` (Error Path)
| Column Name | Type | Purpose / Description |
| :--- | :--- | :--- |
| `event_id` | string | Unique event identifier (e.g. `EVT-P2-20261001-6930`) |
| `created_at` | string | ISO-8601 event timestamp |
| `workflow_name` | string | Constant: `"VenturelyHub Outbound Generation"` |
| `phase` | string | Constant: `"Phase 2 - Outbound Generation"` |
| `event_type` | string | Operational category (e.g. `"MISSING_ORGANIZATION_NAME"`) |
| `status` | string | Status: `"VALIDATION_FAILED"` |
| `test_fixture` | boolean | `true` if synthetic input; `false` otherwise |
| `message` | string | Human-readable error description |
| `raw_event_json` | string | Complete error response JSON string |

---

## 7. Verification & Runtime Execution Summary

### Live n8n Runtime Executions (Verified in Runtime, Ollama Qwen3 8B & Google Sheets)
| Run | Execution ID | Input Entity / Test Category | Pipeline Behavior | Ollama Qwen3 8B Status | Google Sheets Result | Direct Response | Overall Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Test 1** | `13` | `iCreate` (Incubator / Accelerator) | Validated, normalized, generated 4 emails + 3 LinkedIn touches + pitch via Qwen3 8B, validated claims, appended to sheet. | Executed `qwen3:8b` (92.2s, 1486 tokens) | **Appended 1 row** to `Outbound Generation Outputs` (`EXEC-P2-20261001-7306`) | Complete structured package (`PENDING_REVIEW`) | **VERIFIED PASS** |
| **Test 2** | `14` | `KiteMetrics AI` (Startup) | Validated startup profile, generated tailored SaaS engineering package, validated claims, appended to sheet. | Executed `qwen3:8b` (87.8s, 1517 tokens) | **Appended 1 row** to `Outbound Generation Outputs` (`EXEC-P2-20261001-9827`) | Complete structured package (`PENDING_REVIEW`) | **VERIFIED PASS** |
| **Test 3** | `15` | `Apex Logistics` (Enterprise) | Validated enterprise profile, generated logistics technology audit package, validated claims, appended to sheet. | Executed `qwen3:8b` (87.8s, 1487 tokens) | **Appended 1 row** to `Outbound Generation Outputs` (`EXEC-P2-20261001-4927`) | Complete structured package (`PENDING_REVIEW`) | **VERIFIED PASS** |
| **Test 4** | `16` | `StealthCo Labs` (Limited Public Info) | Conservative tone applied, avoided unsupported claims, generated needs assessment package, appended to sheet. | Executed `qwen3:8b` (89.1s, 1518 tokens) | **Appended 1 row** to `Outbound Generation Outputs` (`EXEC-P2-20261001-6452`) | Complete structured package (`PENDING_REVIEW`) | **VERIFIED PASS** |
| **Test 5** | `17` | Missing Org Name `""` (Validation Failure) | `ROUTE_Validation` branched to FALSE, formatted system event row, appended to `System Events`, returned structured error. | Skipped (routed to error branch before LLM) | **Appended 1 row** to `System Events` (`EVT-P2-20261001-6930`); 0 rows to outputs tab | Standardized error payload (`VALIDATION_FAILED`) | **VERIFIED PASS** |
| **Test 6** | `18` | `Example Corp` (Synthetic Domain) | Preserved `test_fixture: true`, generated outbound package with cautious claims, appended to sheet. | Executed `qwen3:8b` (87.4s, 1492 tokens) | **Appended 1 row** to `Outbound Generation Outputs` (`EXEC-P2-20261001-1065`) with `test_fixture: true` | Complete structured package (`PENDING_REVIEW`) | **VERIFIED PASS** |

---

## 8. Status & Credentials
- **Live n8n Workflow ID:** `vhOutboundGen001`
- **Total Nodes:** 16 nodes (14 functional execution nodes + 2 sticky documentation notes)
- **Active State:** `INACTIVE` (`active: false`)
- **Connected Services & Verification Status:**
  - **Local Ollama Outbound Engine (`n8n-nodes-base.httpRequest`):** Connected to local Ollama instance at `http://127.0.0.1:11434/api/chat` running `qwen3:8b` (8.2B parameter model). **AUTHENTICATION_VERIFIED / ZERO-CREDENTIAL** (Local native inference on MacBook Air M4).
  - **Google Sheets OAuth2 API (`googleSheetsOAuth2Api`):** Credential ID `JQRGvtvkfjEPF9WK` (`srisaikirantambalkar@gmail.com`) — **AUTHENTICATION_VERIFIED** for Google Sheets API; live writes verified to spreadsheet `VenturelyHub` (`15__ZAea7cXNS0U3sd-GzsTuZWSFMIgomM_EHPjUr5N4`).
  - **Gmail / LinkedIn:** **NOT USED**. Strictly prohibited in Phase 2.
