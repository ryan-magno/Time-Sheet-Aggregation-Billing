# ✅ TODO — Time Sheet Aggregation & Billing Project

> Last updated: February 25, 2026
>
> Legend: 🔴 = Blocker (must be done before I can proceed) | 🟡 = Needed soon | 🟢 = Can be done later

---

## 🚨 YOUR Tasks (Blockers for Me)

These are things **only you** can do. I cannot proceed to building workflows until these are done.

### 🔴 BU Survey — Ask Each Business Unit Head (CRITICAL BLOCKER)

> [!CAUTION]
> Concerns review revealed that the plan was built from **one BU's perspective**. BFS already confirmed they don't use the SOW process, and each client has unique timesheet templates. You **must survey all BUs** before I build — otherwise we risk building a system that works for one BU and breaks for the rest.

**For EACH of your 10 clients, ask the BU head:**

#### Timesheet Questions
1. **What timesheet template/format does this client require?** (Excel, Google Sheet, client-provided template, web portal, paper form?)
2. **Can we get a sample copy of the template?** (Even a blank one — I need to see column names, layout, date format)
3. **Where are the completed timesheets submitted/stored?** (Google Drive folder? Emailed? Uploaded to client portal? Submitted via JIRA?)
4. **Who fills out the timesheets — our resources directly, or does a PM compile them?**
5. **Do resources also log time in JIRA for this client, or only in the client's template?** (Need to know which sources to collect from per client)

#### Contract / Rate Questions
6. **Is there a signed SOW or Contract Order for this client?** (Yes/No)
7. **If yes — where is it stored?** (Google Drive `/Contracts/`? Another system? Client's system?)
8. **If no SOW — where do billing rates, max hours, and billing terms come from?** (Email? Verbal agreement? Internal doc? Another format?)
9. **What currency is this client billed in?**
10. **What is this client's billing period?** (Calendar month? Mid-month? Bi-weekly? Custom?)
11. **Are there overtime terms? If so, what's the multiplier and does it require client pre-approval?**

#### Process Questions
12. **Who is the Account Manager / Point of Contact for billing of this client?** (Name + email — for discrepancy notifications)
13. **What is the client's finance team email?** (Where invoices get sent)
14. **Does this client have any special billing requirements?** (e.g., specific invoice format, PO numbers, additional reports required)

> [!TIP]
> **Shortcut:** You could create a Google Form with these questions and send it to all 10 BU heads at once. I can help draft the form if you want.

---

### 🔴 IT / Admin Questions (Ask IT Team)

- [ ] 🔴 **Odoo version + hosting** — What version? (16/17/18?) Cloud or self-hosted? Community or Enterprise?
   - Without this, I cannot build any Odoo integration. This was flagged as a high-severity concern.
- [ ] 🔴 **Odoo active modules** — Which of these are installed: `project`, `timesheet`, `hr`, `account`, `sale`?
- [ ] 🔴 **Odoo API test** — Can IT run a simple test API call (`GET /web/dataset/call_kw` for `res.partner`) to confirm auth + connectivity?
- [ ] 🔴 **HRIS API documentation** — Request the API spec from IT. Specifically:
   - Endpoint URL for fetching attendance/timelog by employee and date range
   - Authentication method (API key? OAuth? Username/password?)
   - Response field schema (what fields come back?)
   - Leave-type categories available (sick, vacation, etc.)
- [ ] 🔴 **Who is authorized to verify contract extractions?** — Need names/emails for the verification webhook.

### 🔴 Setup Blockers

- [ ] 🔴 **Attach your skeleton n8n workflows** — You said you'll provide these
- [ ] 🔴 **Set up JIRA Cloud API credentials in n8n**
- [ ] 🔴 **Set up HRIS/Odoo API credentials in n8n**
- [ ] 🔴 **Set up Azure OpenAI API key in n8n** (+ Gemini as fallback)

### ✅ Already Done

- [x] ✅ n8n environment — self-hosted
- [x] ✅ Google OAuth credentials in n8n — set up
- [x] ✅ Google Drive folder structure — exists

### 🟡 Needed During Phase 1–2

- [ ] 🟡 **Provide sample SOW/Contract PDFs** — One from each BU that uses SOWs (redacted OK)
- [ ] 🟡 **Provide sample timesheets** — One sample from each client's unique template
- [ ] 🟡 **Confirm JIRA project keys** per client
- [ ] 🟡 **Provide list of resources (employees)** — Names, IDs, which system(s) they log time in
- [ ] 🟡 **Confirm leave type → billing category mapping** — Review the mapping I proposed:

| Leave Type | Billing Category | Correct? |
|---|---|---|
| Sick Leave | Non-billable | ✅ / ❌ |
| Vacation | Non-billable | ✅ / ❌ |
| Client Holiday | Non-billable | ✅ / ❌ |
| Work From Home | Billable | ✅ / ❌ |
| Unpaid Leave | Non-billable | ✅ / ❌ |
| *(add any missing types)* | | |

- [ ] 🟡 **Confirm HRIS conflict resolution rule** — When HRIS says "on leave" but JIRA has logged hours, should we: (A) Flag as discrepancy for human review? (B) Always trust HRIS? (C) Something else? *(My recommendation: A)*

### 🟡 Needed During Phase 3 (Waiting on HR)

- [ ] 🟡 **Get billing statement template from HR** — Or I can draft one for review
- [ ] 🟡 **Get invoice numbering convention from HR**
- [ ] 🟡 **Get auto-approval threshold from HR** — Currently proposed at $20,000
- [ ] 🟡 **Provide digital signature image** — Image stamp confirmed
- [ ] 🟡 **Provide bank/payment details** for invoice footer

### 🟢 Can Be Done Later (Phase 4–5)

- [ ] 🟢 **Set up client email contacts** — Finance team emails for each client (10 clients)

---

## 🛠️ MY Tasks (What I Will Build)

### Phase 1 — Foundation, Contract Ingestion & Odoo
- [ ] Design Contract Registry schema w/ `human_verified`, `verified_by`, `verified_at`, `confidence_score`, `billing_status`, `processing_lock`, `intake_path`, multi-currency columns
- [ ] Design **Client Template Registry** schema (per-client timesheet config)
- [ ] Create **Audit Log** (Google Sheet)
- [ ] Build **Workflow 1: Contract Ingestion** — Path A (Drive trigger → Azure OpenAI → Sheet → webhook verification button)
- [ ] Build **Workflow 1: Contract Ingestion** — Path B (Manual entry / Google Form)
- [ ] Build **Sub-WF B: Contract Data Extraction** w/ `human_verified` gate
- [ ] Write LLM prompt for SOW/CO PDF extraction
- [ ] Build sanity checks (flag rates >2x or <0.5x average for role)
- [ ] Build webhook-based verification flow (button in email → update sheet + log)
- [ ] Set up Odoo API integration (stub w/ Google Sheet if Odoo details still TBD)
- [ ] Add audit logging to WF1
- [ ] Create test fixtures (dummy contract PDFs)
- [ ] Test Workflow 1 end-to-end (both paths)

### Phase 2 — Timesheet Collection & Validation
- [ ] Build **Sub-WF A: Timesheet Collection** — JIRA Cloud source
- [ ] Build **Sub-WF A: Timesheet Collection** — Per-client template branch (w/ Template Registry field mapping)
- [ ] Build **Sub-WF A: Timesheet Collection** — AI-powered dynamic extraction for unusual formats
- [ ] Build **Sub-WF A: Timesheet Collection** — Scanned docs + OCR (Azure OpenAI / Gemini)
- [ ] Build **Sub-WF A: Timesheet Collection** — HRIS/Odoo source (attendance + leave records)
- [ ] Build **HRIS cross-reference logic** — conflict detection, `HRIS_CONFLICT` discrepancy type
- [ ] Build **leave type → billing category** mapping configuration
- [ ] Add handwritten detection → mandatory human review gate
- [ ] Add **retry logic** per API branch (3 retries, exponential backoff)
- [ ] Add **graceful degradation** (continue with available sources if one fails)
- [ ] Add **LLM cost guard** (max 20 OCR/AI calls per billing run → pause + notify)
- [ ] Build **schema validation** on Sub-WF A input/output
- [ ] Build data merging & deduplication logic
- [ ] Build validation logic: date normalization, hour limits, name standardization
- [ ] Integrate Philippine holiday API for working-day calculations
- [ ] Handle variable billing periods per client
- [ ] Build billable/non-billable categorization
- [ ] Add audit logging to Sub-WF A
- [ ] Create test fixtures (dummy timesheet data — one per client template type)
- [ ] Test Sub-WF A with each data source

### Phase 3 — Billing Computation & Invoice Generation
- [ ] Build **idempotency check** (Billing Run ID + status tracking → prevent double-billing)
- [ ] Build **per-client processing lock** (with 2-hour max timeout + auto-release)
- [ ] Build **catch-up logic** (`<= today` filter for missed billing days)
- [ ] Build **manual trigger guardrails** (client status display + re-run confirmation)
- [ ] Build discrepancy check code nodes (4 checks + `HRIS_CONFLICT` from Step 6)
- [ ] Build billing computation code node (Step 7) — multi-currency
- [ ] Build **schema validation** on Sub-WF C input/output
- [ ] Design Google Docs billing template (or adapt HR's template)
- [ ] Build **Sub-WF C: Invoice Generation** (template fill → PDF export → image stamp)
- [ ] Add audit logging to billing pipeline + Sub-WF C
- [ ] Build **daily run summary email** (after all clients processed)
- [ ] Test billing math against manual calculations
- [ ] Test invoice PDF output
- [ ] Test idempotency (re-run should not double-bill)
- [ ] Test per-client processing lock (Client A stuck should not block Client B)

### Phase 4 — Notifications, Approvals & Delivery
- [ ] Build **Workflow 3: Discrepancy Resolution** w/ **48-hour webhook timeout**
- [ ] Build **3-tier escalation** (48hr reminder → 96hr escalate → 144hr auto-flag delayed)
- [ ] Build **webhook-button approval flow** (approve/reject buttons in email, NOT email reply parsing)
- [ ] Build client email delivery (Gmail + attachments)
- [ ] Build Odoo invoice logging
- [ ] Add audit logging to WF3
- [ ] Test notification emails (formatting, links)
- [ ] Test webhook button approval flow
- [ ] Test webhook timeout + escalation paths

### Phase 5 — Polish & Production
- [ ] Comprehensive error handling review across all workflows
- [ ] Audit Log review — verify end-to-end traceability for a full billing cycle
- [ ] Build **Workflow 4: Payment Tracking** (optional)
- [ ] Write configuration/setup documentation
- [ ] End-to-end production test with redacted real data
- [ ] Handoff documentation

---

## 📝 Notes / Decisions Log

| Date | Decision | By |
|---|---|---|
| 2026-02-19 | Project plan created | Agent |
| 2026-02-19 | User answered all 19 questions. Key: self-hosted n8n, Azure OpenAI/Gemini (not Claude), Odoo from Phase 1, OCR for scans, multi-currency, variable billing periods, 10 clients one-at-a-time, email approvals, image stamp signature | User |
| 2026-02-19 | Plan updated to reflect answers. Follow-up questions asked and resolved. | Agent |
| 2026-02-19 | Dual-trigger design for WF2: daily auto-detect + manual client selector. Per-client billing periods stored in Contract Registry. | Both |
| 2026-02-19 | LLM: Azure OpenAI (GPT) primary, Gemini fallback. Handwritten scanned timesheets always require human verification. | User |
| 2026-02-19 | 10 robustness patterns integrated into plan (idempotency, locks, audit trail, schema validation, etc.) | Agent |
| 2026-02-25 | Reviewed 5 concerns + 2 clarifications from friend's review. All valid. Major changes: (1) WF1 now has dual intake paths (Drive + manual/form), (2) Sub-WF A needs per-client Template Registry, (3) HRIS conflict resolution hierarchy defined, (4) Approval/verification switched from email reply to webhook buttons, (5) Processing lock changed to per-client with 2hr timeout, (6) Daily run summary email added. BU survey questions added to TODO as critical blocker. | Both |
| 2026-02-25 | `CLARIFICATION.md` content incorporated into plan + TODO. File can be deleted. | Agent |

---

## 🔄 Current Status

**Status:** 🔴 Blocked — BU survey + Odoo/HRIS details required before building
**Next step:** You survey BU heads (questions above), confirm Odoo/HRIS details with IT, then I begin Phase 1

---

## ✅ Resolved Decisions

| # | Decision | Answer |
|---|---|---|
| 1 | **LLM provider** | Azure OpenAI (GPT) primary, Gemini as fallback |
| 2 | **Client processing** | Dual-trigger: daily auto-detect (checks `Next Billing Date`) + manual client selector |
| 3 | **Billing periods** | Per-client, stored in Contract Registry. Types: calendar month, mid-month, bi-weekly, custom (N weeks) |
| 4 | **Odoo scope** | Read + Write from Phase 1. *(Odoo version/hosting TBD — BLOCKER)* |
| 5 | **Scanned timesheets** | Can be PDFs, images, handwritten. Varies in format. Handwritten → human verification |
| 6 | **n8n workflows** | You'll provide empty skeleton workflows |
| 7 | **Approval mechanism** | Webhook buttons in email (not email reply parsing — too fragile) |
| 8 | **Contract verification** | Webhook button in email (not manual Google Sheet edit) |
| 9 | **Processing lock** | Per-client, not global. 2-hour max timeout with auto-release. |
| 10 | **Contract intake** | Dual-path: Path A (Drive watcher for SOW PDFs) + Path B (manual entry/form for BUs without SOW) |
| 11 | **HRIS conflict resolution** | HRIS authoritative for attendance, JIRA/Sheets for tasks. Conflicts flagged as `HRIS_CONFLICT` |
| 12 | **Timesheet templates** | Per-client. Client Template Registry stores field mappings per client. |

---

## 📋 Concerns Tracker

Tracks the 5 concerns from `concerns.md` and their resolution status.

| # | Concern | Status | How Resolved |
|---|---|---|---|
| 1 | Email reply approval parsing is fragile | ✅ Resolved | Switched to webhook buttons in email |
| 2 | Odoo version/hosting TBD | ⏳ Pending | Added to BU survey / IT questions above |
| 3 | No admin UI for `human_verified` flag | ✅ Resolved | Webhook button verification + `verified_by`/`verified_at` columns |
| 4 | HRIS cross-referencing logic underspecified | ✅ Resolved | Conflict resolution hierarchy defined in plan. Leave-type mapping pending BU confirmation |
| 5 | Processing lock scope unclear | ✅ Resolved | Changed to per-client locks with 2hr timeout + daily summary email |