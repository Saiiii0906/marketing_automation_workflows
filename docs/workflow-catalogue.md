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
