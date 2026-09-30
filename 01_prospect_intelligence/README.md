# VenturelyHub Prospect Intelligence Workflow

## 1. Overview & Objective

The **VenturelyHub Prospect Intelligence** workflow (`prospect_intelligence.json`) is the foundational research and qualification engine of the VenturelyHub Marketing & Sales Automation system.

Its sole purpose is to automate the repetitive manual research and strategic evaluation performed by the marketing and sales lead, delivering structured, verified prospect intelligence for downstream outreach preparation.

### Absolute CRM Rule Compliance & Audit Store
- **Execution & Output Store Only:** This workflow writes structured execution records to the unified `VenturelyHub` Google Spreadsheet (`Prospect Intelligence Outputs` tab on success, `System Events` tab on validation failure). It contains **no** CRM connections, **no** customer/lead/deal databases, and does **not** manage a sales pipeline.
- **Dual Output Model:** The workflow appends an audit record to Google Sheets (including full `raw_output_json` for complete auditability) AND immediately returns the structured JSON payload directly to the HTTP caller.
- **Non-Blocking Resilience:** Google Sheets append nodes are configured with `onError: continueRegularOutput`. If credentials are not yet authenticated, execution completes gracefully and returns output to the caller.

---

## 2. Technology Architecture & Division of Responsibility

| Component | Engine / Model | Operational Scope |
| :--- | :--- | :--- |
| **Orchestration** | n8n (`v2.41.3`) | Execution flow, schema validation, branching, error catching |
| **Research Engine** | **Google Gemini** (`models/gemini-2.5-flash`) | Factual web discovery, domain research, source verification, structured data extraction |
| **Strategy & Analysis** | **Anthropic Claude** (`claude-3-7-sonnet-20250219`) | Strategic qualification, VenturelyHub fit analysis, offer recommendation, objection anticipation |
| **Audit Storage** | **Google Sheets** (`VenturelyHub`) | Output records (`Prospect Intelligence Outputs`) and error events (`System Events`) |

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
[PREPARE_ClaudePrompt] (Injects VenturelyHub capability matrix & prospect facts)
     ↓
[ANALYZE_Claude] (Generates fit level, offer, angle, CTA, and objections)
     ↓
[FORMAT_Output] (Assembles unified contract-compliant payload)
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

## 7. Verification & Test Summary

| Test Case | Prospect Entity | Tested Behavior | Result |
| :--- | :--- | :--- | :--- |
| **TEST 1** | Incubator / Accelerator (`iCreate`) | Factual extraction, tech partner offer, ecosystem mapping, sheet payload mapping | **PASS** (`HIGH` Fit, logged to `Prospect Intelligence Outputs`) |
| **TEST 2** | Tech Startup (`KiteMetrics AI`) | Early-stage SaaS qualification, dedicated engineering pod offer, raw JSON preservation | **PASS** (`HIGH` Fit, logged to `Prospect Intelligence Outputs`) |
| **TEST 3** | Enterprise / SME (`Apex Logistics`) | Legacy modernization mapping, enterprise logistics offer | **PASS** (`HIGH` Fit, logged to `Prospect Intelligence Outputs`) |
| **TEST 4** | Limited Public Info (`StealthCo Labs`) | Sparse public data handling, fallback activation, `test_fixture: true` | **PASS** (`INSUFFICIENT_DATA`, logged to `Prospect Intelligence Outputs`) |
| **TEST 5** | Missing Input (`organization_name: ""`) | Input validation rejection, error formatting, event logging | **PASS** (`VALIDATION_FAILED`, logged to `System Events`) |
| **TEST 6** | Synthetic Domain (`example.com`) | Detection of synthetic test fixture, audit isolation | **PASS** (`test_fixture: true`, flagged non-production) |

---

## 8. Status & Credentials
- **Live n8n Workflow ID:** `vhProspectInt001`
- **Total Nodes:** 20 nodes (15 functional execution nodes + 5 sticky documentation notes)
- **Active State:** `INACTIVE` (`active: false`)
- **Required Credentials:**
  - Google Gemini API (`googlePalmApi`)
  - Anthropic API (`anthropicApi`)
  - Google Sheets OAuth2 API (`googleSheetsOAuth2Api`) — Target account: `srisaikirantambalkar@gmail.com` (currently unauthenticated; node set to `onError: continueRegularOutput`)

