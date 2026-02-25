## 📋 STEP-BY-STEP BREAKDOWN

### **STEP 1: Submit Signed SOW/Contract**

**What happens in real life:**

- Sales team closes a deal with a client (e.g., ABC Corp needs 2 developers)
- They create a contract (Statement of Work - SOW) that says:
    - "We'll provide John (Senior Dev) at $50/hour, max 160 hours/month"
    - "We'll provide Sarah (Junior Dev) at $30/hour, max 160 hours/month"
- Contract is signed by both parties (Exist and ABC Corp)
- PDF is saved in Google Drive folder: `/Contracts/ABC Corp/SOW_ABC_2024.pdf`

**What n8n will do:**

- **Trigger:** Watch the Google Drive `/Contracts/` folder
- **Action:** When a new PDF appears with "SOW" or "CO" in the filename
- **Store:** Remember the location of this contract for later use

**Why this matters:** The contract is the "source of truth" for billing. It tells us WHO to bill, HOW MUCH per hour, and MAXIMUM hours allowed.

---

### **STEP 2: Collect Timesheet Data**

**What happens in real life:**

- John and Sarah work all month on the project
- Every day, they log their time in different places:
    - John uses JIRA: "Worked 8 hours on Feature X for ABC Corp"
    - Sarah uses Google Sheets: "8 hours - ABC Corp - Development"
    - Sometimes they submit scanned paper timesheets (photos/PDFs)
- At month-end, someone has to open ALL these different systems and download/collect all the data

**Where the data lives:**

```
Google Drive Structure:
📁 Timesheets/
  📁 ABC Corp/
    📄 John_January_2024.xlsx
    📄 Sarah_January_2024_GoogleSheet (link)
    📄 Scanned_Weekly_Timesheet.pdf

```

**What n8n will do:**

1. **Connect to JIRA API**
    - Search for all time entries for January 2024
    - Filter by project = "ABC Corp"
    - Download data in JSON format
2. **Connect to Google Sheets**
    - Open the timesheet Google Sheet
    - Read all rows for January 2024
    - Extract: Date, Resource Name, Hours, Project
3. **Connect to Google Drive**
    - Find scanned PDFs/images
    - Download them for processing
4. **Connect to HRIS (employee.exist.com)**
    - Get official employee work hours
    - This is already synced with Odoo
    - Use this as the "master record" to verify against

**Output at this stage:**

```json
[
  {
    "source": "JIRA",
    "resource": "John Smith",
    "date": "2024-01-15",
    "hours": 8,
    "project": "ABC Corp",
    "billable": true
  },
  {
    "source": "Google Sheets",
    "resource": "Sarah Jones",
    "date": "2024-01-15",
    "hours": 8,
    "project": "ABC Corp",
    "billable": true
  }
]

```

**Challenges n8n handles:**

- John might write "John Smith" in JIRA but "J. Smith" in Google Sheets, so use ID
- Dates might be formatted differently (01/15/2024 vs 2024-01-15)
- Some entries might be duplicates
- Need to collect from 3+ different sources and combine into ONE dataset

---

### **STEP 3: Submit Validated Timesheet Data**

**What happens in real life:**

- Project Manager (Jude) reviews all collected timesheets
- Checks for obvious errors:
    - "Wait, John logged 25 hours on January 15th? That's impossible!"
    - "Sarah logged hours on Sunday? Was that approved?"
- Marks which hours are "billable" (client pays) vs "non-billable" (client doesn't pay)
    - Billable: Development work, meetings with client, bug fixing
    - Non-billable: Sick leave, training, internal meetings, holidays
- Saves the cleaned-up data back to Google Drive as "validated"

**What n8n will do:**

1. **Take collected data from Step 2**
2. **Run validation checks:**
    - Hours per day should be ≤ 24 (catch typos like "80 hours" instead of "8 hours")
    - No work logged on weekends/holidays (unless OT is approved)
    - All entries have: Date, Resource, Hours, Project
    - Resource names are standardized ("John Smith" everywhere, not "J.Smith")
3. **Categorize billable vs non-billable:**
    - Look at "Activity Type" field
    - "Development" = billable
    - "Training" = non-billable
    - "Sick Leave" = non-billable
4. **Create clean dataset:**

```json
[
  {
    "resource": "John Smith",
    "date": "2024-01-15",
    "hours": 8,
    "project": "ABC Corp",
    "billable": true,
    "activity": "Development"
  },
  {
    "resource": "John Smith",
    "date": "2024-01-16",
    "hours": 8,
    "project": "Internal",
    "billable": false,
    "activity": "Training"
  }
]

```

1. **Save to Google Drive:** `/Validated_Timesheets/ABC_Corp_January_2024_Validated.json`

**Why this step exists:** Raw data is messy. We need clean, standardized data before we can compare it to contracts and do math.

---

### **STEP 4: Access Contract Information**

**What happens in real life:**

- Billing person opens the contract PDF
- Manually reads through it to find:
    - Page 3: "John Smith - Senior Developer - $50/hour"
    - Page 3: "Maximum 160 hours per month"
    - Page 5: "Rates are exclusive of 12% VAT"
    - Page 6: "Overtime approved at 130% of standard rate"
- Writes this down in a spreadsheet to use for calculations

**What n8n will do:**

1. **Get contract from Step 1:** `/Contracts/ABC Corp/SOW_ABC_2024.pdf`
2. **Send to Claude API (LLM):**

```jsx
// n8n HTTP Request Node
POST https://api.anthropic.com/v1/messages
{
  "model": "claude-sonnet-4-20250514",
  "messages": [{
    "role": "user",
    "content": [
      {
        "type": "document",
        "source": {
          "type": "base64",
          "media_type": "application/pdf",
          "data": "<base64_encoded_pdf>"
        }
      },
      {
        "type": "text",
        "text": `Extract the following information from this contract:

        For each resource mentioned:
        - Full name
        - Role/position
        - Hourly or daily rate
        - Currency
        - Maximum hours per month

        Also extract:
        - Is VAT included in rates or added on top?
        - VAT percentage
        - Overtime terms (if any)
        - Contract start and end dates
        - Client name
        - Project name

        Return as JSON format.`
      }
    ]
  }]
}

```

1. **Claude returns structured data:**

```json
{
  "client": "ABC Corp",
  "project": "Web Development",
  "contract_period": {
    "start": "2024-01-01",
    "end": "2024-12-31"
  },
  "resources": [
    {
      "name": "John Smith",
      "role": "Senior Developer",
      "rate": 50,
      "rate_type": "hourly",
      "currency": "USD",
      "max_hours_per_month": 160
    },
    {
      "name": "Sarah Jones",
      "role": "Junior Developer",
      "rate": 30,
      "rate_type": "hourly",
      "currency": "USD",
      "max_hours_per_month": 160
    }
  ],
  "vat_terms": {
    "rate_is_vat_inclusive": false,
    "vat_percentage": 12
  },
  "overtime_terms": {
    "allowed": true,
    "rate_multiplier": 1.3,
    "requires_client_approval": true
  }
}

```

1. **Store this data** in n8n workflow memory for next steps

**Why we use AI here:**

- Every contract looks different
- Rates might be in a table, or in a paragraph, or in a bullet list
- Some say "$50 per hour", others say "fifty dollars hourly"
- AI can understand all these variations

**What if AI makes a mistake?**

- First few runs: Human reviews the extracted data
- If extraction looks wrong, we can improve the prompt
- Once confident, let it run automatically
- Always flag unusual values (rate = $500/hr? Seems wrong!)

---

### **STEP 5: Extract Billable and Non-Billable Hours**

**What happens in real life:**

- Using the validated timesheet, someone manually:
    - Filters for "ABC Corp" project only
    - Counts billable hours for John in January
    - Counts non-billable hours for John in January
    - Repeats for Sarah
    - Puts everything in a spreadsheet

**What n8n will do:**

1. **Take validated timesheet data from Step 3**
2. **Group by Resource and Billable/Non-Billable:**

```jsx
// n8n Code Node
const timesheets = $input.all();

// Group data
const summary = {};

timesheets.forEach(entry => {
  const key = entry.json.resource;

  if (!summary[key]) {
    summary[key] = {
      resource: key,
      project: entry.json.project,
      billable_hours: 0,
      non_billable_hours: 0,
      billable_breakdown: [],
      non_billable_breakdown: []
    };
  }

  if (entry.json.billable) {
    summary[key].billable_hours += entry.json.hours;
    summary[key].billable_breakdown.push({
      date: entry.json.date,
      hours: entry.json.hours,
      activity: entry.json.activity
    });
  } else {
    summary[key].non_billable_hours += entry.json.hours;
    summary[key].non_billable_breakdown.push({
      date: entry.json.date,
      hours: entry.json.hours,
      activity: entry.json.activity
    });
  }
});

return Object.values(summary);

```

1. **Output:**

```json
[
  {
    "resource": "John Smith",
    "project": "ABC Corp",
    "billable_hours": 152,
    "non_billable_hours": 8,
    "billable_breakdown": [
      {"date": "2024-01-02", "hours": 8, "activity": "Development"},
      {"date": "2024-01-03", "hours": 8, "activity": "Development"},
      // ... more entries
    ],
    "non_billable_breakdown": [
      {"date": "2024-01-16", "hours": 8, "activity": "Training"}
    ]
  },
  {
    "resource": "Sarah Jones",
    "project": "ABC Corp",
    "billable_hours": 160,
    "non_billable_hours": 0
  }
]

```

**Why this step:** We need monthly totals to compare against contracts and calculate bills.

---

### **STEP 6: Initial Verification & Discrepancy Flagging**

**This is the MOST IMPORTANT step** - where we catch errors before billing the client!

**What happens in real life:**

- Billing person manually checks everything:
    - "John's rate in timesheet = $50? ✓ Matches contract"
    - "John logged 152 hours? ✓ Under max of 160"
    - "Wait... Sarah logged 160 billable + 0 non-billable = 160 total. But January has 22 working days × 8 hrs = 176 hours. Where are the missing 16 hours? She didn't take any leave?"
    - "John logged 152 billable + 8 non-billable = 160 total. That's only 20 days. Did he take 2 days off?"

**What n8n will do - 4 major checks:**

### **Check 1: Rate & Max Hours Verification**

```jsx
// n8n Code Node
const extractedHours = $('Step5').all(); // From Step 5
const contractInfo = $('Step4').first().json; // From Step 4

const discrepancies = [];

extractedHours.forEach(resource => {
  // Find this resource in contract
  const contractResource = contractInfo.resources.find(
    r => r.name === resource.json.resource
  );

  if (!contractResource) {
    discrepancies.push({
      type: "RESOURCE_NOT_IN_CONTRACT",
      resource: resource.json.resource,
      message: `${resource.json.resource} logged hours but is not in the contract!`
    });
    return;
  }

  // Check max hours
  if (resource.json.billable_hours > contractResource.max_hours_per_month) {
    discrepancies.push({
      type: "HOURS_EXCEED_MAX",
      resource: resource.json.resource,
      logged_hours: resource.json.billable_hours,
      max_hours: contractResource.max_hours_per_month,
      excess: resource.json.billable_hours - contractResource.max_hours_per_month,
      message: `${resource.json.resource} logged ${resource.json.billable_hours} hours, exceeding contract max of ${contractResource.max_hours_per_month} by ${resource.json.billable_hours - contractResource.max_hours_per_month} hours.`
    });
  }
});

```

### **Check 2: Total Hours Reconciliation**

```jsx
// Calculate expected monthly hours
const workingDaysInJanuary = 22; // Exclude weekends/holidays
const expectedHoursPerDay = 8;
const expectedMonthlyHours = workingDaysInJanuary * expectedHoursPerDay; // 176 hours

extractedHours.forEach(resource => {
  const totalLogged = resource.json.billable_hours + resource.json.non_billable_hours;

  if (totalLogged !== expectedMonthlyHours) {
    discrepancies.push({
      type: "TOTAL_HOURS_MISMATCH",
      resource: resource.json.resource,
      total_logged: totalLogged,
      expected_hours: expectedMonthlyHours,
      difference: expectedMonthlyHours - totalLogged,
      message: `${resource.json.resource} logged ${totalLogged} total hours. Expected ${expectedMonthlyHours} hours (${workingDaysInJanuary} days × 8 hrs). Missing ${expectedMonthlyHours - totalLogged} hours of logs.`
    });
  }
});

```

### **Check 3: Contract Allocation Check**

```jsx
// Standard allocation is 160 hours/month for full-time
const standardAllocation = 160;

extractedHours.forEach(resource => {
  if (resource.json.billable_hours !== standardAllocation) {
    const variance = resource.json.billable_hours - standardAllocation;
    const percentage = (variance / standardAllocation * 100).toFixed(1);

    discrepancies.push({
      type: "ALLOCATION_DEVIATION",
      resource: resource.json.resource,
      logged_hours: resource.json.billable_hours,
      standard_allocation: standardAllocation,
      variance: variance,
      percentage: percentage,
      message: `${resource.json.resource} logged ${resource.json.billable_hours} billable hours vs standard allocation of ${standardAllocation} hours (${variance > 0 ? '+' : ''}${variance} hours, ${percentage}%).`
    });
  }
});

```

### **Check 4: Overtime Validation**

```jsx
extractedHours.forEach(resource => {
  const contractResource = contractInfo.resources.find(
    r => r.name === resource.json.resource
  );

  if (resource.json.billable_hours > contractResource.max_hours_per_month) {
    const overtimeHours = resource.json.billable_hours - contractResource.max_hours_per_month;

    if (contractInfo.overtime_terms.allowed) {
      discrepancies.push({
        type: "OVERTIME_NEEDS_APPROVAL",
        resource: resource.json.resource,
        overtime_hours: overtimeHours,
        overtime_rate: contractResource.rate * contractInfo.overtime_terms.rate_multiplier,
        message: `${resource.json.resource} has ${overtimeHours} hours of overtime. Verify client approval exists. Overtime rate: $${contractResource.rate * contractInfo.overtime_terms.rate_multiplier}/hr (${contractInfo.overtime_terms.rate_multiplier}x standard).`
      });
    } else {
      discrepancies.push({
        type: "OVERTIME_NOT_ALLOWED",
        resource: resource.json.resource,
        overtime_hours: overtimeHours,
        message: `${resource.json.resource} exceeded max hours by ${overtimeHours}, but contract does not allow overtime. Cannot bill these hours.`
      });
    }
  }
});

```

**Example output:**

```json
{
  "discrepancies": [
    {
      "type": "TOTAL_HOURS_MISMATCH",
      "resource": "Sarah Jones",
      "total_logged": 160,
      "expected_hours": 176,
      "difference": 16,
      "message": "Sarah Jones logged 160 total hours. Expected 176 hours (22 days × 8 hrs). Missing 16 hours of logs."
    }
  ],
  "clean_data": [
    {
      "resource": "John Smith",
      "billable_hours": 152,
      "non_billable_hours": 8,
      "status": "OK"
    }
  ]
}

```

**What n8n does with discrepancies:**

- **IF** discrepancies found → Flag them, segregate that resource's data
- **IF** no discrepancies → Mark as "verified and ready for billing"

---

### **STEP 7: Compute Totals**

**What happens in real life:**

- For each resource, calculate: Hours × Rate = Amount
- Add up all resources
- Calculate VAT
- Get grand total

**What n8n will do:**

```jsx
// n8n Code Node
const verifiedData = $('Step6').first().json.clean_data;
const contractInfo = $('Step4').first().json;

const billingCalculations = [];
let subtotal = 0;

verifiedData.forEach(resource => {
  // Find rate from contract
  const contractResource = contractInfo.resources.find(
    r => r.name === resource.resource
  );

  // Calculate amount
  const amount = resource.billable_hours * contractResource.rate;
  subtotal += amount;

  billingCalculations.push({
    resource_name: resource.resource,
    role: contractResource.role,
    hours_billable: resource.billable_hours,
    rate: contractResource.rate,
    currency: contractResource.currency,
    total_amount: amount
  });
});

// Calculate VAT
const vatRate = contractInfo.vat_terms.vat_percentage / 100;
const vatAmount = contractInfo.vat_terms.rate_is_vat_inclusive ? 0 : subtotal * vatRate;
const grandTotal = subtotal + vatAmount;

return [{
  json: {
    billing_period: "January 2024",
    client: contractInfo.client,
    project: contractInfo.project,
    line_items: billingCalculations,
    subtotal: subtotal,
    vat_rate: contractInfo.vat_terms.vat_percentage,
    vat_amount: vatAmount,
    grand_total: grandTotal,
    currency: contractInfo.resources[0].currency
  }
}];

```

**Output:**

```json
{
  "billing_period": "January 2024",
  "client": "ABC Corp",
  "project": "Web Development",
  "line_items": [
    {
      "resource_name": "John Smith",
      "role": "Senior Developer",
      "hours_billable": 152,
      "rate": 50,
      "currency": "USD",
      "total_amount": 7600
    },
    {
      "resource_name": "Sarah Jones",
      "role": "Junior Developer",
      "hours_billable": 160,
      "rate": 30,
      "currency": "USD",
      "total_amount": 4800
    }
  ],
  "subtotal": 12400,
  "vat_rate": 12,
  "vat_amount": 1488,
  "grand_total": 13888,
  "currency": "USD"
}

```

**Math breakdown:**

- John: 152 hrs × $50 = $7,600
- Sarah: 160 hrs × $30 = $4,800
- Subtotal: $12,400
- VAT (12%): $12,400 × 0.12 = $1,488
- **Grand Total: $13,888**

---

### **STEP 8: Generate Draft Billing Statement**

**What happens in real life:**

- Someone opens a Word/Excel template
- Manually types in all the numbers from calculations
- Formats it nicely
- Saves as PDF

**What n8n will do:**

1. **Use a Google Docs template** (pre-made by your team)

Template looks like:

```
═══════════════════════════════════════
         EXIST SOFTWARE LABS INC.
              BILLING STATEMENT
═══════════════════════════════════════

Client: {{client_name}}
Project: {{project_name}}
Billing Period: {{billing_period}}
Date: {{invoice_date}}
Invoice #: {{invoice_number}}

───────────────────────────────────────
RESOURCE DETAILS
───────────────────────────────────────

{{#each line_items}}
Resource: {{resource_name}}
Role: {{role}}
Hours: {{hours_billable}} hrs
Rate: {{currency}} {{rate}}/hr
Amount: {{currency}} {{total_amount}}

{{/each}}

───────────────────────────────────────
SUMMARY
───────────────────────────────────────
Subtotal:        {{currency}} {{subtotal}}
VAT ({{vat_rate}}%):      {{currency}} {{vat_amount}}
                 ─────────────────────
GRAND TOTAL:     {{currency}} {{grand_total}}

───────────────────────────────────────
Payment Terms: Net 30 days
Bank Details: [Bank info here]

```

1. **n8n fills in the template:**

```jsx
// n8n Google Docs Node
// 1. Copy template
// 2. Replace placeholders with actual data from Step 7
// 3. Export as PDF

```

1. **Output:** `Draft_Invoice_ABC_Corp_Jan2024.pdf`

**What it looks like:**

```
═══════════════════════════════════════
         EXIST SOFTWARE LABS INC.
              BILLING STATEMENT
═══════════════════════════════════════

Client: ABC Corp
Project: Web Development
Billing Period: January 1-31, 2024
Date: February 1, 2024
Invoice #: INV-2024-001

───────────────────────────────────────
RESOURCE DETAILS
───────────────────────────────────────

Resource: John Smith
Role: Senior Developer
Hours: 152 hrs
Rate: USD 50/hr
Amount: USD 7,600

Resource: Sarah Jones
Role: Junior Developer
Hours: 160 hrs
Rate: USD 30/hr
Amount: USD 4,800

───────────────────────────────────────
SUMMARY
───────────────────────────────────────
Subtotal:        USD 12,400.00
VAT (12%):       USD 1,488.00
                 ─────────────────────
GRAND TOTAL:     USD 13,888.00

```

1. **Save to Google Drive:** `/Billing/Drafts/ABC_Corp_Jan2024_Draft.pdf`

---

### **STEP 9: Notify Account Manager (if discrepancies)**

**What happens in real life:**

- If there were errors in Step 6, someone emails the Account Manager
- "Hey Sarah, John's timesheet has issues, please check"

**What n8n will do:**

1. **Check if Step 6 found any discrepancies**

```jsx
// n8n IF Node
if (discrepancies.length > 0) {
  // Go to notification flow
} else {
  // Skip to Step 10
}

```

1. **If discrepancies exist, send email:**

```jsx
// n8n Gmail Node
To: account.manager@exist.com
Subject: [ACTION REQUIRED] Billing Discrepancies - ABC Corp - January 2024

Body:

```

**Email content:**

```
Hi Sarah,

The automated billing system has detected discrepancies in the January 2024
billing for ABC Corp that require your review:

DISCREPANCY SUMMARY:
─────────────────────────────────────────────────────────

1. TOTAL_HOURS_MISMATCH
   Resource: Sarah Jones
   Issue: Logged 160 total hours, expected 176 hours (22 working days × 8 hrs)
   Missing: 16 hours of logs

   Possible reasons:
   - Did Sarah take 2 days off without logging leave?
   - Were there days she forgot to submit timesheets?
   - Was she on another project for those hours?

   Action needed: Verify with Sarah and update timesheets

─────────────────────────────────────────────────────────

LINKS:
📄 View Timesheet: [Link to Google Sheet]
📄 View Contract: [Link to SOW PDF]
📄 Draft Bill: [Link to Draft PDF]

WHAT TO DO:
1. Review the discrepancies above
2. Correct timesheets in the system
3. Reply to this email when corrections are complete
4. The system will regenerate the final bill

DEADLINE: Please complete by February 3, 2024 (2 days)

If you have questions, contact the PMO team.

Best regards,
Automated Billing System

```

**Attachments:**

- Screenshot of the specific timesheet rows with issues
- Relevant section of the contract
1. **Wait for Account Manager to fix:**
- Account Manager contacts Sarah Jones
- Sarah updates her timesheet (adds missing leave days)
- Account Manager marks issue as "resolved" in system
- n8n detects resolution and proceeds

---

### **STEP 10: Generate Final Billing Statement**

**What happens in real life:**

- After all corrections are made, create the final invoice
- Add digital signature
- Mark as "FINAL - DO NOT EDIT"

**What n8n will do:**

1. **Get corrected data:**
    - If there were discrepancies: Use corrected timesheet data
    - If no discrepancies: Use original verified data
2. **Re-run calculations** (Steps 5-7) with corrected data
3. **Generate final document:**

```jsx
// n8n Google Docs Node
// 1. Use same template as draft
// 2. Add "FINAL" watermark
// 3. Add invoice number
// 4. Add digital signature field

```

1. **Apply digital signature:**

```jsx
// Option 1: Use Google Docs built-in signature
// Option 2: Use DocuSign API
// Option 3: Use simple image signature stamp

// For simplicity, let's use image signature
// Add signature image to bottom of document:
// "Digitally signed by: [Authorized Signer Name]"
// "Date: February 1, 2024"
// [Signature image]

```

1. **Export as PDF:** `Final_Invoice_ABC_Corp_Jan2024.pdf`
2. **Save to Google Drive:** `/Billing/Final/2024/February/ABC_Corp_Jan2024_Final.pdf`
3. **Mark as "Locked"** - no further edits allowed

---

### **STEP 11: Second Review & Final Approval**

**What happens in real life:**

- Before sending to client, a senior person (Account Manager or VP) does final check
- "Everything looks correct? Numbers add up? Signature present? OK, approved!"

**What n8n will do:**

**Option A: Automatic approval** (if no discrepancies and under certain threshold)

```jsx
// n8n IF Node
if (grandTotal < 20000 && discrepancies.length === 0) {
  // Auto-approve
  approvalStatus = "AUTO_APPROVED";
} else {
  // Send for manual approval
}

```

**Option B: Manual approval** (if amount is high or had discrepancies)

```jsx
// n8n Email Node
To: vp.sales@exist.com
Subject: [APPROVAL NEEDED] Final Invoice - ABC Corp - $13,888

Body:
"Please review and approve the final invoice for ABC Corp.

Invoice Details:
- Client: ABC Corp
- Period: January 2024
- Amount: $13,888
- Status: Ready to send

View Invoice: [Link to PDF]

Reply with:
APPROVE - to send to client
REJECT - to send back for revision
"

```

**Approval methods:**

1. **Email reply:** VP replies "APPROVE"
2. **Web interface:** VP clicks "Approve" button on a simple webpage
3. **Slack:** VP clicks "Approve" in Slack message

**n8n waits for approval:**

```jsx
// n8n Webhook Node (waiting for approval)
// Once "APPROVE" received, proceed to Step 12
// If "REJECT", go back to Step 9 (notify for corrections)

```

---

### **STEP 12: Send Notification Email to Client**

**What happens in real life:**

- Someone emails the invoice to the client
- Attaches all necessary documents
- Uses professional language
- Marks invoice as "SENT" in system

**What n8n will do:**

1. **Gather all documents:**

```jsx
// Files to attach:
const attachments = [
  '/Billing/Final/2024/February/ABC_Corp_Jan2024_Final.pdf',           // Main invoice
  '/Contracts/ABC Corp/SOW_ABC_2024.pdf',                              // Contract reference
  '/Reports/ABC_Corp_Jan2024_NonBillable.pdf',                        // Non-billable hours report
  '/Reports/ABC_Corp_Jan2024_Consolidated.pdf'                        // Consolidated report
];

```

1. **Generate professional email:**

```jsx
// n8n Gmail Node

To: finance@abccorp.com
CC: project.manager@abccorp.com, billing@exist.com
Subject: Exist Billing ending January 31, 2024

Body:

```

**Email content:**

```
Dear ABC Corp Finance Team,

Please find attached the billing statement for services rendered during the
period of January 1-31, 2024.

INVOICE SUMMARY:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Invoice Number:     INV-2024-001
Billing Period:     January 1-31, 2024
Invoice Date:       February 1, 2024
Total Amount Due:   USD $13,888.00
Payment Due Date:   March 2, 2024 (Net 30)
━━━━━━━━━━━━━━

```

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

RESOURCES BILLED THIS PERIOD:
• John Smith (Senior Developer):  152 hours @ $50/hr = $7,600
• Sarah Jones (Junior Developer): 160 hours @ $30/hr = $4,800

Subtotal: $12,400.00
VAT (12%): $1,488.00
TOTAL: $13,888.00

ATTACHED DOCUMENTS:

1. Final Invoice (INV-2024-001.pdf)
2. Statement of Work Reference (SOW_ABC_2024.pdf)
3. Non-Billable Hours Report (for your reference)
4. Consolidated Monthly Report

PAYMENT DETAILS:
Bank Name: [Your Bank]
Account Number: [Account #]
Swift Code: [Swift]
Reference: INV-2024-001

If you have any questions regarding this invoice, please don't hesitate
to contact us at billing@exist.com or call +63-XXX-XXXX.

Thank you for your continued partnership.

Best regards,

[Digital Signature]
Exist Software Labs Inc.
Billing Department
billing@exist.com
www.exist.com

```

3. **Send email with all attachments**

4. **Log in system:**
```javascript
// Update Odoo
{
  "invoice_id": "INV-2024-001",
  "client": "ABC Corp",
  "amount": 13888,
  "status": "SENT",
  "sent_date": "2024-02-01",
  "due_date": "2024-03-02",
  "sent_by": "n8n_automation"
}

```

1. **Set reminder:**

```jsx
// n8n Schedule Node
// Set reminder for 25 days from now (5 days before due date)
// If payment not received, send reminder email

```

---

## 🎯 COMPLETE FLOW SUMMARY

**What just happened (the whole process):**

1. ✅ Contract uploaded → n8n detected it
2. ✅ Timesheets collected from JIRA, Google Sheets, scans
3. ✅ Data validated and cleaned
4. ✅ AI extracted contract details (rates, max hours, VAT terms)
5. ✅ Hours categorized (billable vs non-billable)
6. ✅ Everything cross-checked for errors
    - Rates match? ✓
    - Hours within limits? ✓
    - Total hours add up? ✓
7. ✅ Calculations done (hours × rates + VAT)
8. ✅ Draft invoice created
9. ✅ If errors: Account Manager notified and fixed them
10. ✅ Final invoice generated with signature
11. ✅ VP approved
12. ✅ Client received invoice via email

**Time saved:**

- Manual process: 2-3 days
- Automated: 2 hours (mostly waiting for approvals)

**Errors prevented:**

- Wrong rates
- Miscalculations
- Missing hours
- Forgot to bill overtime
- Wrong VAT

---

## 🔧 WHAT YOU'LL BUILD IN N8N

Here's what the n8n canvas will look like:

```
[Google Drive Trigger: New Contract]
    → [Store Contract Location]

[Schedule Trigger: 1st of Month]
    → [Collect JIRA Data]
    → [Collect Google Sheets Data]
    → [Collect Scanned Timesheets]
    → [Merge All Timesheet Data]
    → [Validate & Clean Data]
    → [Load Contract from Step 1]
    → [HTTP Request: Claude API - Extract Contract Data]
    → [Code Node: Group Hours by Resource]
    → [Code Node: Run Discrepancy Checks]
    → [IF Node: Discrepancies?]
        YES → [Gmail: Notify Account Manager]
            → [Webhook: Wait for Corrections]
        NO → [Continue]
    → [Code Node: Calculate Totals]
    → [Google Docs: Generate Draft]
    → [Code Node: Re-calculate with Corrections]
    → [Google Docs: Generate Final Invoice]
    → [IF Node: Need Manual Approval?]
        YES → [Gmail: Request VP Approval]
            → [Webhook: Wait for Approval]
        NO → [Auto Approve]
    → [Gather Attachments]
    → [Gmail: Send to Client]
    → [Update Odoo]
    → [Set Payment Reminder]

```

---

**What you're actually doing:**

1. **Connecting systems** - Make JIRA talk to Google Sheets talk to Claude API
2. **Writing logic** - "If this, then that" rules in Code nodes
3. **Handling errors** - What if data is missing? What if API fails?
4. **Testing thoroughly** - Run with test data first, compare to manual process
5. **Documenting** - Write down what each node does for future reference

**Skills you'll learn:**

- API integration (JIRA, Google, Claude)
- Data transformation (JSON, CSV, PDF)
- Workflow logic (if/else, loops, data mapping)
- Error handling
- Testing & debugging

**Common mistakes to avoid:**

- Don't assume data is perfect (always validate)
- Don't skip error handling (what if Gmail is down?)
- Don't hardcode values (use variables for dates, rates, etc.)
- Don't forget logging (how will you debug if something fails?)
- Don't test with real client data (use dummy data first)