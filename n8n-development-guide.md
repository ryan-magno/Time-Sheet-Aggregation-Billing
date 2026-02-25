# AI Coding Assistant — n8n Workflow Architecture Guide

> This file is the single source of truth for how any AI coding assistant (GitHub Copilot, Claude, Cursor, etc.) should build, structure, and reason about n8n workflows in this environment. Follow every rule unless the user explicitly overrides it for a specific session.

---

## 0. Before You Build Anything

### 0.1 Version Check
Before writing any workflow, expression, or node configuration, ask the user:

> "What version of n8n are you running? (Check: **Settings → About**)"

Expression syntax, AI node structure, and available nodes differ between versions. If the user doesn't know, default to the **latest stable n8n syntax** and flag any version-sensitive parts of your output.

### 0.2 Clarify Before Building
If any requirement is ambiguous — data source, trigger frequency, auth method, output destination — stop and ask. Ask **one clarifying question at a time**, scoped to the most critical ambiguity. Do not interrogate the user across five questions before writing a single node. Resolve blockers in priority order: trigger → data shape → destination → error handling → format.

Do not assume. Do not infer. If it is not stated, ask.

---

## 1. Node Naming Convention (Mandatory)

Every node must be renamed to a descriptive, action-oriented name before the workflow is considered complete. Never leave default names like `HTTP Request`, `Set`, `IF`, or `Code`.

**Format:** `Verb_Object` in PascalCase with underscores.

```
✅ Fetch_User_Profile
✅ Send_Welcome_Email
✅ Parse_Webhook_Payload
✅ Check_Record_Exists
✅ Transform_To_Invoice_Schema
✅ Log_Error_To_Sheet

❌ HTTP Request 2
❌ Set1
❌ IF
❌ Code
❌ Webhook
```

**Why this matters:** The explicit node referencing syntax (`$('Node Name').item.json.field`) breaks silently when node names are ambiguous or duplicated. Descriptive names make expressions readable, debuggable, and maintainable.

When you name a node, also rename any downstream expressions that reference it immediately. Never leave a renamed node with stale references in other nodes.

---

## 2. Explicit Node Referencing (Mandatory)

Never rely on implicit data passing (`$json`, `$input`) in workflows with more than two nodes. Always reference data by its originating node name.

**Use modern n8n expression syntax exclusively:**

```
✅ {{ $('Fetch_User_Profile').item.json.email }}
✅ {{ $('Parse_Webhook_Payload').first().json.userId }}
✅ {{ $('Get_All_Orders').all() }}  ← inside Code node only

❌ {{ $json.email }}
❌ {{ $node["HTTP Request"].json["email"] }}  ← legacy syntax, forbidden
❌ json.email
```

### 2.1 When to Use `.item` vs `.first()` vs `.all()`

| Method | Use When |
|---|---|
| `.item.json.field` | Accessing data from the current loop item in a paired node |
| `.first().json.field` | You need a single value from a previous node's full output (e.g., a config lookup) |
| `.all()` | Inside a **Code node** when you need to iterate over every item a previous node returned |

Using `.first()` on a multi-item node when you needed `.all()` is one of the most common silent data loss errors in n8n. When in doubt, log the output count before processing.

---

## 3. Integration Discovery Order (MCP → Native → HTTP)

When the user requests an integration, action, or external service connection, always follow this lookup chain:

1. **Check MCP first.** Query available MCP tools and servers in the current environment. If a native MCP capability exists, use it. Do not build an HTTP workaround when an MCP tool is available.
2. **Native n8n nodes second.** If no MCP tool exists, check for a native n8n node (e.g., Slack, Google Sheets, Notion, Airtable). Default to the native node.
3. **HTTP Request node last.** If neither MCP nor a native node exists, fall back to the HTTP Request node. When doing so, always:
   - Structure the endpoint, method, headers, and body explicitly.
   - Reference authentication via n8n Credentials (never inline).
   - Add a node to validate the HTTP response code before continuing.

---

## 4. AI Agents — When and How

Default to n8n AI Agent nodes (LangChain Agent, Tools Agent, Chains) instead of brittle IF/Switch logic when **any** of the following are true:

- There are more than 2 conditional branches based on content or intent
- The output format or structure is unpredictable from the input
- The task requires tool-calling (web search, database lookup, API call based on context)
- The routing decision depends on semantic meaning, not a simple value match
- Data extraction is fuzzy (e.g., "pull the date from this email")

Do **not** default to AI Agents for deterministic tasks: value lookups, format transforms, sending a fixed payload to a single endpoint, or simple data mapping. Use a Set, Code, or native node for those.

### 4.1 Idempotency for AI Agent Workflows

Any workflow triggered by an AI Agent or loop **must** include an idempotency check before performing write operations (sending emails, inserting records, calling APIs).

Before executing the action, check:
- Has this record already been processed? (Check a database, Airtable field, or Google Sheet column)
- Has this email already been sent? (Check a sent log or a "processed" tag)
- Does this record already exist? (Query before inserting)

If the check confirms the action was already completed, branch to a `Skip_Already_Processed` node and stop. Never assume a workflow runs exactly once.

---

## 5. Error Handling — Three-Tier System

"Build for robustness" means applying the right error handling tool at the right level. There are three tiers:

### Tier 1 — Global: Error Trigger Node
Add an `Error_Trigger` workflow for every production workflow. This catches unhandled errors from the entire workflow and routes them to a notification (Slack, email) or a log.

- Every production workflow should have a companion error-handling workflow connected via the Error Trigger node.
- Log the workflow name, node name, error message, and timestamp at minimum.

### Tier 2 — Node Level: Continue On Fail
For nodes that call external services (HTTP Request, Send Email, database writes), ask the user:

> "Should this node continue the workflow if it fails, or halt entirely?"

If continuing: enable **Continue On Fail** on that node and add a downstream IF node to check `$('Node_Name').item.json.error` and branch accordingly.

If halting: leave Continue On Fail off and rely on the global Error Trigger.

Never silently enable Continue On Fail without a downstream check. A node that fails silently and continues is worse than a node that halts.

### Tier 3 — Data Level: Validate Before Processing
Before any processing node that assumes data exists, add a validation check:

```javascript
// In a Code node — always validate before iterating
const items = $('Fetch_Records').all();
if (!items || items.length === 0) {
  return [{ json: { status: 'no_data', skipped: true } }];
}
```

Never assume an upstream node returned data. Always account for empty arrays and null values.

---

## 6. Webhook Workflows — Respond Early

If a workflow is triggered by a Webhook node and contains any long-running operations (AI calls, multiple API requests, database writes), follow this pattern:

1. Place the `Respond_to_Webhook` node **immediately after** the Webhook node or after minimal validation.
2. Send an acknowledgement response (`200 OK`, `{ "status": "received" }`).
3. Continue the rest of the workflow asynchronously after the response.

Failing to do this causes the caller to wait for the entire workflow execution and time out. Most webhook callers expect a response within 5–10 seconds.

---

## 7. Sub-Workflows for Modularity

If a workflow exceeds **10–15 nodes**, or contains a distinct logical unit that could be reused (e.g., "enrich a contact," "send a notification," "validate an invoice"), extract it into a sub-workflow using the `Execute Workflow` node.

Sub-workflow rules:
- Name sub-workflows descriptively: `Sub_Enrich_Contact`, `Sub_Send_Notification`
- Pass data explicitly via the Execute Workflow node's input. Never rely on shared global state.
- Sub-workflows must handle their own errors internally and return a structured response (`{ success: true/false, data: ..., error: ... }`)
- The parent workflow checks the sub-workflow's response before continuing.

Sub-workflows keep the main workflow readable, testable, and reusable. A 40-node monolith is a maintenance disaster.

---

## 8. n8n Data Structure Rules

n8n passes all data as an **array of items**:

```javascript
[ { "json": { ... } }, { "json": { ... } } ]
```

Every Code node output **must** match this structure. No exceptions.

```javascript
// ✅ Correct
return items.map(item => ({
  json: {
    processed: true,
    userId: item.json.userId,
    timestamp: new Date().toISOString()
  }
}));

// ❌ Wrong — will break downstream nodes
return { userId: 123 };
return [{ userId: 123 }];
```

When iterating over data from a previous node inside a Code node:

```javascript
const allItems = $('Fetch_Orders').all();

return allItems.map(item => ({
  json: {
    orderId: item.json.id,
    total: item.json.amount * 1.1
  }
}));
```

---

## 9. Security — Credentials and Secrets

**Never hardcode API keys, bearer tokens, passwords, or secrets** in any node — HTTP Request headers, Code nodes, or expressions.

Always:
- Create a named **n8n Credential** for any service requiring authentication.
- Reference environment variables for secrets not covered by native credential types: `{{ $env["MY_SECRET_KEY"] }}`
- If the user provides a raw API key in the chat, do not embed it in the workflow. Instruct them to store it as a credential or environment variable first.

If you generate an HTTP Request node with authentication, always leave the auth field pointing to a credential placeholder and add a comment instructing the user to configure it.

---

## 10. Development & Testing Mode

When building a new workflow, always structure it for safe testing before production activation.

### 10.1 Manual Trigger During Development
Add a **Manual Trigger** node alongside the production trigger (Webhook, Schedule, etc.) during development. This allows testing without activating the live trigger. Remove or disable the Manual Trigger before handoff.

### 10.2 Static Test Data
For any node that receives external data (Webhooks, API responses, database reads), provide a **static test data block** in a comment or Set node for development use:

```json
// Static test payload for Webhook — remove before production
{
  "userId": "test-001",
  "email": "testuser@example.com",
  "plan": "pro",
  "createdAt": "2025-01-01T00:00:00Z"
}
```

Pin this data in n8n's node execution panel during development so downstream nodes process predictable inputs. When handing off the workflow, flag which pinned data needs to be unpinned.

### 10.3 Development Checklist
Before marking a workflow ready for production, confirm:

- [ ] All nodes renamed to descriptive `Verb_Object` names
- [ ] All expressions use explicit node referencing (`$('Node_Name').item.json.field`)
- [ ] Static/pinned test data removed or unpinned
- [ ] Manual Trigger node disabled or removed
- [ ] Error handling configured on all external-facing nodes
- [ ] Credentials stored in n8n Credential manager — no hardcoded secrets
- [ ] Idempotency check in place for any write or send operation
- [ ] Webhook `Respond_to_Webhook` placed correctly (if applicable)
- [ ] Sub-workflows extracted if node count exceeds 15
- [ ] Workflow tested end-to-end with real data at least once

---

## 11. Agentic Logging Protocol

Track every significant session outcome — both failures and successes. The model is the constant. The user's input is the variable. Focus on the variable.

### Log Storage
```
./logs/errors/error-[ID].md
./logs/successes/success-[ID].md
./logs/metadata.json          ← ID counters live here
./logs/templates/error-template.md
./logs/templates/success-template.md
```

Logs must be **generic** — structured so they are useful in future sessions, not tied to a specific project's terminology.

### 11.1 When to Log an Error
Log when any of the following occur:
- AI-generated workflow produces incorrect output
- An expression or node reference breaks during execution
- The user had to significantly correct or rewrite AI output
- A hallucination occurred (invented node name, wrong syntax, fabricated API behavior)
- A workflow was built that violated any rule in this document

**Error Logging Steps:**
1. Review the conversation and identify what went wrong.
2. Ask the user **5–8 pointed questions** focused on their behavior — prompt structure, context state, what they assumed vs. what they said.
3. Trace and quote the **exact triggering prompt** verbatim.
4. Read `./logs/metadata.json` to get the next error ID.
5. Write the log to `./logs/errors/error-[ID].md` using `./logs/templates/error-template.md`.
6. Increment the error counter in `metadata.json`.

Be critical. The user is logging this to grow. Do not soften the diagnosis.

### 11.2 When to Log a Success
Log when any of the following occur:
- A complex workflow built correctly on the first attempt
- An AI Agent integration worked without correction
- An expression or data mapping was unusually precise and clean
- The user's prompting produced notably fast or high-quality output

**Success Logging Steps:**
1. Review the conversation and identify the specific win.
2. Ask the user **4–6 questions** to extract exactly why it worked.
3. Trace and quote the **exact triggering prompt** verbatim.
4. Read `./logs/metadata.json` to get the next success ID.
5. Write the log to `./logs/successes/success-[ID].md` using `./logs/templates/success-template.md`.
6. Increment the success counter in `metadata.json`.

### 11.3 metadata.json Structure
```json
{
  "last_updated": "2025-01-01T00:00:00Z",
  "errors": {
    "count": 0,
    "last_id": "error-000"
  },
  "successes": {
    "count": 0,
    "last_id": "success-000"
  }
}
```

---

## 12. Quick Reference — Rules Summary

| Rule | Non-Negotiable? |
|---|---|
| Ask n8n version before building | Yes |
| Rename all nodes to `Verb_Object` format | Yes |
| Use explicit node referencing only | Yes |
| Follow MCP → Native → HTTP chain | Yes |
| Never hardcode secrets | Yes |
| Code node output must be array of items | Yes |
| Add idempotency check to all write/send ops | Yes |
| Respond to Webhook early in long workflows | Yes |
| Suggest sub-workflows at 10–15 node threshold | Yes |
| Add static test data during development | Yes |
| Log errors and successes per protocol | Yes |