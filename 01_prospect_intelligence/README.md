# VenturelyHub Prospect Intelligence Workflow

## 1. Overview & Objective

The **VenturelyHub Prospect Intelligence** workflow (`prospect_intelligence.json`) is the foundational research and qualification engine of the VenturelyHub Marketing & Sales Automation system.

Its sole purpose is to automate the repetitive manual research and strategic evaluation performed by the marketing and sales lead, delivering structured, verified prospect intelligence for downstream outreach preparation.

### Absolute CRM Rule Compliance
- **Zero Persistent Storage:** This workflow is a stateless, pure-transformation data pipeline. It contains **no** database nodes, **no** Google Sheets sinks, **no** CRM connections, and creates **no** persistent prospect records.
- **Output-Only Model:** Input payloads are ingested via webhook, validated, researched, analyzed, and immediately returned as a structured JSON object to the caller.

---

## 2. Technology Architecture & Division of Responsibility

| Component | Engine / Model | Operational Scope |
| :--- | :--- | :--- |
| **Orchestration** | n8n (`v2.41.3`) | Execution flow, schema validation, branching, error catching |
| **Research Engine** | **Google Gemini** (`models/gemini-2.5-flash`) | Factual web discovery, domain research, source verification, structured data extraction |
| **Strategy & Analysis** | **Anthropic Claude** (`claude-3-7-sonnet-20250219`) | Strategic qualification, VenturelyHub fit analysis, offer recommendation, objection anticipation |

---

## 3. Workflow Architecture & Pipeline Sequence

```
INPUT PAYLOAD
     ↓
[TRIGGER_ProspectInput] (Webhook POST /webhook/prospect-intelligence)
     ↓
[VALIDATE_Input] (Trims whitespace, enforces required fields, sanitizes URLs)
     ↓
[ROUTE_Validation] (If: isValid === true)
     ├── FALSE → [FORMAT_ErrorOutput] → RETURN ERROR OBJECT
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
RETURN STRUCTURED INTELLIGENCE
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

## 6. Verification & Test Summary

| Test Case | Prospect Entity | Tested Behavior | Result |
| :--- | :--- | :--- | :--- |
| **TEST 1** | Incubator / Accelerator (`iCreate`) | Factual extraction, tech partner offer, ecosystem mapping | **PASS** (`HIGH` Fit) |
| **TEST 2** | Tech Startup (`KiteMetrics AI`) | Early-stage SaaS qualification, dedicated engineering pod offer | **PASS** (`HIGH` Fit) |
| **TEST 3** | Enterprise / SME (`Apex Logistics`) | Legacy modernization mapping, enterprise logistics offer | **PASS** (`HIGH` Fit) |
| **TEST 4** | Limited Public Info (`StealthCo Labs`) | Sparse public data handling, fallback activation | **PASS** (`INSUFFICIENT_DATA`) |
| **TEST 5** | Missing Input (`organization_name: ""`) | Graceful rejection, structured error payload returned | **PASS** (`VALIDATION_FAILED`) |

---

## 7. Status & Credentials
- **Live n8n Workflow ID:** `vhProspectInt001`
- **Active State:** `INACTIVE` (`active: false`)
- **Credentials Required:**
  - Google Gemini API (`googlePalmApi`)
  - Anthropic API (`anthropicApi`)
