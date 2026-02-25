## What the BU is Telling You

### Statement 1: "Each client requires us to use their specific timesheet template"

**What this means:**
The assumption in the current plan is that timesheets come from standardized sources — JIRA, a uniform Google Sheets format, or scanned docs. The BU is saying that in reality, **each client dictates their own timesheet format**. Client A has their own Excel template, Client B has a web portal, Client C has a PDF form, etc. The resources (employees) fill out whatever template the client requires, not a standard internal one.

**What changes in the plan:**

- **Sub-WF A (Timesheet Collection) needs a complete rethink.** You can no longer assume a consistent schema from Google Sheets. Each client's template will have different column names, date formats, row structures, and layouts. You can't write one normalization script — you need either a per-client normalization config or a more robust AI-powered extraction that figures out the structure dynamically.
- **The "normalize to standard schema" step becomes significantly harder.** Right now the plan treats normalization as a simple field mapping. With arbitrary client templates, this becomes an OCR/AI extraction problem even for Excel files — not just scanned PDFs.
- **You'll need a template registry.** For each client, you need to store: what their timesheet looks like, where the relevant fields are (resource name, date, hours, project code), and how to extract them. This is additional metadata to maintain.
- **Testing complexity multiplies.** Instead of testing one timesheet format, you now need test fixtures for each of the 10 clients' templates.

---

### Statement 2: Removing the SOW/Contract Step for BFS

**What this means:**
The BU (specifically BFS — likely the Banking & Financial Services unit) is saying that the process of **formally submitting a signed SOW/contract to a Google Drive folder** is not how they operate. They either don't use SOWs, use a different contract format, store contracts elsewhere, or the contract intake process is handled outside of this workflow entirely. They're also uncertain whether other BUs follow the SOW submission process at all.

**What changes in the plan:**

- **Workflow 1 (Contract Ingestion) cannot be assumed universal.** It was designed around a specific trigger: a PDF with "SOW" or "CO" in the filename appearing in a Google Drive folder. If BFS doesn't follow this, WF1 either doesn't trigger for them, or triggers incorrectly.
- **Contract data sourcing becomes per-BU.** For BFS, if there's no SOW in Google Drive, where does the billing rate and max hours information come from? Someone has to answer this — otherwise the `human_verified` contract data gate in WF2 will always block BFS from billing.
- **The `human_verified` flag strategy may need to change for BFS.** If contract data is entered manually (not extracted from a PDF), the workflow needs a different intake path — perhaps a Google Form or manual entry directly into the Contract Registry, with the `human_verified` flag set to `true` by default upon entry.
- **"I'm not sure if applicable to other BUs" is a red flag.** This suggests the original requirements were written from one BU's perspective and haven't been validated across all business units. You likely have inconsistency across all 10 clients, not just BFS.

---

## The Bigger Picture

These two statements together reveal a **fundamental assumption gap** in the current plan:

The plan was designed as if Exist has one standardized process across all clients and BUs. The reality is that **each client (and possibly each BU) has its own process, templates, and contract handling**. This means:

| What the Plan Assumed | What's Actually True |
|---|---|
| One standard timesheet format | Each client has their own template |
| All contracts are SOW/CO PDFs in Google Drive | At least one BU (BFS) doesn't use this process |
| One universal workflow for all clients | May need per-client or per-BU configuration |

**Immediate action needed before building anything further:**

You need to survey all BUs and answer two questions for each client:
1. What does their timesheet template look like, and where is it submitted?
2. Where does contract/rate information come from, and how is it stored?

Without this, you risk building a system that works for one BU and breaks for the rest.