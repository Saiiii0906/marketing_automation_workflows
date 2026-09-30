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
|  - Dual-LLM intelligence engine (Gemini + Claude)           |
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
      │             ANTHROPIC CLAUDE (STRATEGY ENGINE)         │
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
- **Trigger Layer:** Webhook receiving target organization metadata.
- **Validation Layer:** Enforces input hygiene and required organization name.
- **Research Layer:** Gemini-powered search with built-in tools (`googleSearch`, `urlContext`).
- **Structuring Layer:** Normalizes facts, verifies data density, triggers defensive fallbacks if information is sparse.
- **Analysis Layer:** Claude-powered commercial fit analysis mapping against VenturelyHub core service domains.
- **Output Layer:** Returns unified contract-compliant intelligence payload.

---

### 4. Safety & Human Governance Protocol

1. **No Autonomous Outbound:** Automation creates structured intelligence and recommendations. It does not send unreviewed external emails or commit commercial terms.
2. **Execution State:** All production workflows remain **INACTIVE** in the n8n runtime until explicitly authorized.
3. **Audit Trail:** Every workflow version and schema modification is version-controlled in Git and reconciled with the local n8n runtime.
