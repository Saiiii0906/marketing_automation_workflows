# VenturelyHub Prospect Intelligence Workflow

## 1. Overview & Objective

The **VenturelyHub Prospect Intelligence** workflow (`prospect_intelligence.json`) is the foundational research and qualification engine of the VenturelyHub Marketing & Sales Automation system.

Its sole purpose is to automate the repetitive manual research and strategic evaluation performed by the marketing and sales lead, delivering structured, verified prospect intelligence for downstream outreach preparation.

### Absolute CRM Rule Compliance & Audit Store
- **Execution & Output Store Only:** This workflow writes structured execution records to the unified `VenturelyHub` Google Spreadsheet (`Prospect Intelligence Outputs` tab on success, `System Events` tab on validation failure). It contains **no** CRM connections, **no** customer/lead/deal databases, and does **not** manage a sales pipeline.
- **Persistent Output Architecture:** Google Sheets operates as a persistent audit/output layer while the workflow continues to return the structured response to the caller.
- **Traceability & Idempotency Boundary:** `execution_id` provides unique execution-level traceability. The current append-only audit layer does not guarantee duplicate suppression across manual/replayed executions.
- **Non-Blocking Resilience:** Google Sheets append nodes are configured with `onError: continueRegularOutput`. If transient issues occur, execution completes gracefully and returns output to the caller.

---

## 2. Technology Architecture & Division of Responsibility

| Component | Engine / Model | Operational Scope |
| :--- | :--- | :--- |
| **Orchestration** | n8n (`v2.41.3`) | Execution flow, schema validation, branching, error catching |
| **Research Engine** | **Google Gemini** (`models/gemini-2.5-flash`) | Factual web discovery, domain research, source verification, structured data extraction |
| **Strategy & Analysis** | **Ollama / Qwen3 8B** (`qwen3:8b` via `http://127.0.0.1:11434/api/chat`) | Strategic qualification, VenturelyHub fit analysis, offer recommendation, objection anticipation |
| **Audit Storage** | **Google Sheets** (`VenturelyHub`) | Persistent output records (`Prospect Intelligence Outputs`) and error events (`System Events`) |

---

## 3. Workflow Architecture & Pipeline Sequence

```
INPUT PAYLOAD
     ↓
[TRIGGER_ProspectInput] (Webhook POST /webhook/prospect-intelligence)
     ↓
[VALIDATE_Input] (Trims whitespace, enforces required fields, sanitizes URLs, detects test fixtures)
     ↓
[ROUTE_Validation] (If: isValid === true)
     ├── FALSE → [FORMAT_ErrorOutput] 
     │                ↓
     │           [GOOGLE_SHEETS_Append_SystemEvent] (Appends to 'System Events' tab)
     │                ↓
     │           [RETURN_ErrorOutput] (Returns structured error JSON to caller)
     ↓ TRUE
[PREPARE_GeminiPrompt] (Injects grounding & source-discipline rules)
     ↓
[RESEARCH_Gemini] (Google Search + URL context enabled, JSON output)
     ↓
[STRUCTURE_Research] (Parses, validates research schema, verifies data density)
     ↓
[PREPARE_StrategyPrompt] (Injects VenturelyHub capability matrix & prospect facts)
     ↓
[ANALYZE_Strategy_Ollama] (Calls local Ollama API qwen3:8b for fit, offer, angle, CTA, objections)
     ↓
[FORMAT_Output] (Parses Ollama JSON, deterministic fallback if needed, assembles payload)
     ↓
[VALIDATE_SheetPayload] (Prepares execution row, preserves raw_output_json, generates execution_id)
     ↓
[GOOGLE_SHEETS_Append_ProspectIntelligence] (Appends to 'Prospect Intelligence Outputs' tab)
     ↓
[RETURN_Output] (Returns structured intelligence JSON to caller)
```

---

## 4. Input Specification

The workflow accepts POST requests at `/webhook/prospect-intelligence` with JSON payloads:

### Fields
| Field | Type | Required | Description | Example |
| :--- | :--- | :--- | :--- | :--- |
| `organization_name` | string | **Yes** | Official name of target entity | `"iCreate"` |
| `website` | string | Optional | Official domain URL | `"https://www.icreate.org.in"` |
| `linkedin_url` | string | Optional | Company or institution LinkedIn URL | `"https://linkedin.com/company/icreate"` |
| `organization_type` | string | Optional | Category: `incubator`, `startup`, `enterprise`, etc. | `"incubator"` |
| `city` | string | Optional | City of headquarters / primary campus | `"Ahmedabad"` |
| `state` | string | Optional | State or province | `"Gujarat"` |
| `country` | string | Optional | Country | `"India"` |
| `user_notes` | string | Optional | Contextual notes from user | `"Focusing on DeepTech and IoT"` |

---

## 5. Output Specification

### Success Response (`status: "SUCCESS"`)
```json
{
  "prospect_profile": {
    "organization_name": "iCreate (International Centre for Entrepreneurship and Technology)",
    "website": "https://www.icreate.org.in",
    "type": "Incubator/Accelerator",
    "location": "Ahmedabad, Gujarat, India",
    "description": "Autonomous centre of excellence facilitating tech innovation...",
    "focus_areas": ["DeepTech", "IoT", "GreenTech", "Healthcare", "Electric Vehicles"],
    "industries": ["Cleantech", "Hardware/Embedded Systems", "Robotics"],
    "programs": ["Sparkup Idea Fund", "iCreate Incubation Program", "EV Angel Fund"],
    "technology_focus": ["AI/ML", "Embedded Systems", "Cloud Integration", "IoT Sensors"]
  },
  "research": {
    "relevant_activities": ["Annual startup demo day", "EV innovation challenge"],
    "partnership_signals": ["Formal MoUs with tech partners", "External mentorship pool"],
    "relevant_contact_roles": ["Head of Incubation", "GM - Technology Partnerships"],
    "sources": ["https://www.icreate.org.in", "https://www.startupindia.gov.in"],
    "research_date": "2026-09-30"
  },
  "venturelyhub_analysis": {
    "fit_level": "HIGH",
    "fit_reasons": [
      "Strong alignment as an incubator ecosystem partner...",
      "Incubated startups require custom software engineering and cloud MVP delivery..."
    ],
    "relevant_services": [
      "Product Engineering",
      "AI Development & AI Agents",
      "Cloud Architecture & DevOps",
      "UI/UX Design"
    ],
    "recommended_offer": "Preferred Technology Partner & Startup Product Clinic",
    "value_proposition": "VenturelyHub can support iCreate cohort founders with rapid MVP engineering...",
    "personalization_angle": "Reference their active cohorts in DeepTech and EV innovation challenges...",
    "recommended_contact_role": "Head of Incubation / GM - Technology Partnerships",
    "suggested_cta": "Propose an exploratory 15-minute call to discuss a co-hosted Technical Product Clinic...",
    "potential_objections": [
      "Startups manage their own technical teams independently",
      "Incubator already has informal technology mentors"
    ],
    "confidence": "HIGH"
  },
  "metadata": {
    "workflow_name": "VenturelyHub Prospect Intelligence",
    "processed_at": "2026-09-30T13:25:28.615Z",
    "status": "SUCCESS"
  }
}
```

### Error Response (`status: "VALIDATION_FAILED"`)
```json
{
  "error": {
    "code": "MISSING_REQUIRED_FIELD",
    "message": "Field organization_name is required and cannot be empty."
  },
  "prospect_profile": {
    "organization_name": "",
    "website": "Not provided",
    "type": "Not specified",
    "location": "Location not specified",
    "description": "Validation failed before research could begin.",
    "focus_areas": [],
    "industries": [],
    "programs": [],
    "technology_focus": []
  },
  "research": {
    "relevant_activities": [],
    "partnership_signals": [],
    "relevant_contact_roles": ["Relevant contact not publicly verified."],
    "sources": [],
    "research_date": "2026-09-30"
  },
  "venturelyhub_analysis": {
    "fit_level": "INSUFFICIENT_DATA",
    "fit_reasons": ["Validation error: Missing required organization name."],
    "relevant_services": [],
    "recommended_offer": "None",
    "value_proposition": "None",
    "personalization_angle": "None",
    "recommended_contact_role": "Relevant contact not publicly verified.",
    "suggested_cta": "None",
    "potential_objections": [],
    "confidence": "LOW"
  },
  "metadata": {
    "workflow_name": "VenturelyHub Prospect Intelligence",
    "processed_at": "2026-09-30T13:25:28.615Z",
    "status": "VALIDATION_FAILED"
  }
}
```

---

## 6. Google Sheets Execution & Audit Schemas

The workflow appends records to the central `VenturelyHub` spreadsheet across two target tabs:

### Tab 1: `Prospect Intelligence Outputs` (Success Path)
| Column Name | Type | Purpose / Description |
| :--- | :--- | :--- |
| `execution_id` | string | Unique execution identifier (e.g. `EXEC-P1-20260930-4821`) |
| `created_at` | string | ISO-8601 execution timestamp |
| `workflow_name` | string | Constant: `"VenturelyHub Prospect Intelligence"` |
| `phase` | string | Constant: `"Phase 1 - Prospect Intelligence"` |
| `status` | string | Execution outcome status: `"SUCCESS"` |
| `test_fixture` | boolean | `true` if synthetic domain/test entity; `false` for genuine prospects |
| `organization_name`| string | Researched organization name |
| `fit_level` | string | Strategic fit rating: `HIGH` \| `MEDIUM` \| `LOW` \| `INSUFFICIENT_DATA` |
| `recommended_offer`| string | Primary tailored VenturelyHub service offer |
| `recommended_contact_role` | string | Recommended public job title for outreach |
| `research_date` | string | Date research was conducted (`YYYY-MM-DD`) |
| `raw_output_json` | string | Complete verbatim JSON output string preserving all nested data |
| `error_message` | string | Empty on success (`""`) |

### Tab 2: `System Events` (Error / Validation Failure Path)
| Column Name | Type | Purpose / Description |
| :--- | :--- | :--- |
| `event_id` | string | Unique event identifier (e.g. `EVT-P1-20260930-3194`) |
| `created_at` | string | ISO-8601 event timestamp |
| `workflow_name` | string | Constant: `"VenturelyHub Prospect Intelligence"` |
| `phase` | string | Constant: `"Phase 1 - Prospect Intelligence"` |
| `event_type` | string | Operational category: `"VALIDATION_FAILURE"` |
| `status` | string | Status: `"VALIDATION_FAILED"` |
| `test_fixture` | boolean | `true` if synthetic input; `false` otherwise |
| `message` | string | Human-readable error description |
| `raw_event_json` | string | Complete error response JSON string |

---

## 7. Verification & Runtime Execution Summary

### A. Live n8n Runtime Executions (Verified in Runtime, Ollama Qwen3 8B & Google Sheets)
| Run | Execution ID | Input Entity / Test Category | Pipeline Behavior | Ollama Qwen3 8B Status | Google Sheets Result | Direct Response | Overall Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Test 1** | `7` | `iCreate` (Incubator / Accelerator) | Full pipeline executed. Grounded research by Gemini, strategic evaluation by Ollama Qwen3 8B, formatted & appended. | Executed `qwen3:8b` (42.3s, 720 tokens) | **Appended 1 row** to `Prospect Intelligence Outputs` (`EXEC-P1-20261001-9067`) | Complete structured JSON with status `SUCCESS` | **VERIFIED PASS** |
| **Test 2** | `8` | `KiteMetrics AI` (Startup) | Startup profile research, Ollama Qwen3 8B strategic evaluation, formatted & appended. | Executed `qwen3:8b` (42.4s, 764 tokens) | **Appended 1 row** to `Prospect Intelligence Outputs` (`EXEC-P1-20261001-8680`) | Complete structured JSON with status `SUCCESS` | **VERIFIED PASS** |
| **Test 3** | `9` | `Apex Logistics` (Enterprise) | Enterprise profile research, Ollama Qwen3 8B strategic evaluation, formatted & appended. | Executed `qwen3:8b` (43.2s, 781 tokens) | **Appended 1 row** to `Prospect Intelligence Outputs` (`EXEC-P1-20261001-8348`) | Complete structured JSON with status `SUCCESS` | **VERIFIED PASS** |
| **Test 4** | `10` | `StealthCo Labs` (Limited Public Info) | Sparse domain research, fallback activation, Ollama Qwen3 8B conservative qualification. | Executed `qwen3:8b` (35.6s, 627 tokens) | **Appended 1 row** to `Prospect Intelligence Outputs` (`EXEC-P1-20261001-3566`) | Complete structured JSON (`INSUFFICIENT_DATA`, `LOW`) | **VERIFIED PASS** |
| **Test 5** | `11` | Missing Org Name `""` (Validation Failure) | `ROUTE_Validation` branched to FALSE, formatted system event row, appended to `System Events`, delivered direct error response. | Skipped (routed to error branch before LLM) | **Appended 1 row** to `System Events` (`EVT-P1-20261001-1401`); 0 rows to outputs tab | Standardized error payload (`VALIDATION_FAILED`) | **VERIFIED PASS** |
| **Test 6** | `12` | `Example Corp` (Synthetic Domain) | Preserved `test_fixture: true`, research & Ollama strategic evaluation completed, appended to sheet. | Executed `qwen3:8b` (39.0s, 701 tokens) | **Appended 1 row** to `Prospect Intelligence Outputs` (`EXEC-P1-20261001-1759`) with `test_fixture: true` | Complete structured JSON with status `SUCCESS` | **VERIFIED PASS** |

---

## 8. Status & Credentials
- **Live n8n Workflow ID:** `vhProspectInt001`
- **Total Nodes:** 20 nodes (15 functional execution nodes + 5 sticky documentation notes)
- **Active State:** `INACTIVE` (`active: false`)
- **Connected Services & Verification Status:**
  - **Google Gemini API (`googlePalmApi`):** Credential ID `9pUlYyCqAwOgAvb8` (`tiwarivivek102006@gmail.com`) — **AUTHENTICATION_VERIFIED** (Active model: `models/gemini-2.5-flash`).
  - **Local Ollama Strategic Engine (`n8n-nodes-base.httpRequest`):** Connected to local Ollama instance at `http://127.0.0.1:11434/api/chat` running `qwen3:8b` (8.2B parameter model). **AUTHENTICATION_VERIFIED / ZERO-CREDENTIAL** (Local native inference on MacBook Air M4).
  - **Google Sheets OAuth2 API (`googleSheetsOAuth2Api`):** Credential ID `JQRGvtvkfjEPF9WK` (`srisaikirantambalkar@gmail.com`) — **AUTHENTICATION_VERIFIED** for Google Sheets API; live writes verified to spreadsheet `VenturelyHub` (`15__ZAea7cXNS0U3sd-GzsTuZWSFMIgomM_EHPjUr5N4`).
  - **Anthropic Claude API (`anthropicApi`):** **REMOVED**. Completely replaced by local Ollama Qwen3 8B. Zero external Claude dependencies remain in the workflow.


