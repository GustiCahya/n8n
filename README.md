# n8n workflow templates: 5 standalone workflows with built-in error handling

Five production-style n8n workflows, each **fully self-contained**: no shared error workflow and no cross-workflow setup. Every file carries its own documentation on the canvas (one yellow overview sticky, white section stickies, a note on every node) and its own **built-in error handler**.

| File | Workflow name | Trigger | Services |
|------|---------------|---------|----------|
| `workflows/01-lead-capture-crm-whatsapp.json` | Capture website leads to a Google Sheets CRM with instant WhatsApp alerts | Webhook `POST /lead-capture` | Google Sheets, WhatsApp Cloud API, Gmail |
| `workflows/02-ai-data-processing-pipeline.json` | Clean, summarize and categorize incoming text with OpenAI into Google Sheets | Webhook `POST /ai-pipeline` | OpenAI, Google Sheets, Gmail |
| `workflows/03-webhook-to-sheets-reliable-ingest.json` | Reliable webhook to Google Sheets ingest with retries and a dead-letter queue | Webhook `POST /ingest` + Schedule (15 min) | Google Sheets, n8n Data Tables, Gmail |
| `workflows/04-typeform-airtable-slack.json` | Route Typeform submissions to Airtable and notify the right owner in Slack | Typeform Trigger | Typeform, Airtable, Slack, Gmail |
| `workflows/05-ai-agent-record-filter.json` | AI agent that filters records and forwards only human-review cases to Slack | Webhook `POST /ai-record-filter` | OpenAI (AI Agent + tools), Google Sheets, Slack, Gmail |

---

## 1. Built-in error handler (in every workflow)

The bottom of each canvas holds an **⚠️ Built-in error handler** section:

```
On Workflow Error (Error Trigger) → Alert Settings → Build Incident Report → Email Alert (Gmail, 3 retries)
                                                                              └─ on failure → Fallback (per workflow)
```

**Why it works without any settings.** When an execution fails and *Settings → Error workflow* is empty, n8n looks for an Error Trigger **inside the same workflow** and runs it as the "internal error workflow". This was verified in n8n's source (`packages/cli/src/execution-lifecycle/execute-error-workflow.ts`). The error run never re-triggers itself (n8n guards on `mode === 'error'`), and manual test runs do not trigger it.

| Workflow | Fallback when the email alert fails | Extra setup |
|---|---|---|
| 01 | **WhatsApp** message to the ops number in *Alert Settings* | ops number |
| 02 | Row in the **`Error Log`** sheet tab | create the tab |
| 03 | `INCIDENT` row in the existing **`ingest_request_log`** Data Table | none |
| 04, 05 | Message to the **Slack** ops channel in *Alert Settings* | invite the bot |

Each fallback reuses a credential the workflow already has, so the only extra credential is Gmail. To use Outlook or SMTP instead, swap out the **Email Alert** node.

**What reaches the handler.** Unexpected crashes, exhausted retries on nodes without an error branch, and deliberate escalations from the *Escalate ·* (Stop and Error) nodes. Escalation messages start with a tag that becomes the email's severity: `[CRITICAL]` = act now, `[WARNING]` = recovered automatically, no tag = `ERROR`.

**Testing the handler.** Open the workflow and execute **On Workflow Error** on its own. In manual mode it emits sample data, which proves the email and the fallback end to end.

## 2. Install one workflow (≈ 10–15 minutes each)

1. *Workflows → Import from File* → choose the JSON.
2. Open the yellow overview sticky (top left) and follow **Setup steps**.
3. Search the workflow for `REPLACE_WITH_` (spreadsheet IDs, WhatsApp phone-number ID, Airtable base/table IDs, Typeform form ID). The nodes named **Config** and **Alert Settings** hold all business settings.
4. Leave *Settings → Error workflow* **empty**, so the built-in handler is used.
5. Run the `curl` test from the first section sticky, then **activate**.

Credentials are deliberately not in the JSON. Every credential field shows a normal dropdown after import.

## 3. Credentials

| Credential type | 01 | 02 | 03 | 04 | 05 |
|---|:-:|:-:|:-:|:-:|:-:|
| Gmail OAuth2 (built-in error handler) | ✓ | ✓ | ✓ | ✓ | ✓ |
| Header Auth (`X-API-Key: <long random secret>`) | | ✓ | ✓ | | ✓ |
| Google Sheets OAuth2 | ✓ | ✓ | ✓ | | ✓ |
| WhatsApp Business Cloud API | ✓ | | | | |
| OpenAI | | ✓ | | | ✓ |
| Typeform API | | | | ✓ | |
| Airtable Personal Access Token (`data.records:write`, `schema.bases:read`) | | | | ✓ | |
| Slack (bot invited to every target channel) | | | | ✓ | ✓ |

01's webhook is public on purpose because it receives a browser form. It is protected by a honeypot field and strict validation. If your form posts from a server, enable Header Auth on it too.

## 4. External resources

**Google Sheets** (put these headers in row 1):

| Workflow | Tab | Columns |
|---|---|---|
| 01 | `Leads` | Lead ID, Submitted At, Name, Email, Phone, Company, Message, Source, UTM Source, UTM Campaign, Status, Alert Status, Alerted At |
| 02 | `Processed` | *(leave empty: headers are written automatically)* |
| 02 | `Error Log` | Incident ID, Occurred At, Severity, Workflow, Failed Node, Error, Execution URL |
| 03 | `Events` | Request ID, Received At, Event, Reference, Customer Email, Amount, Currency, Notes, Stored Via |
| 05 | `Review Log` | Batch ID, Record ID, Reviewed At, Customer Email, Title, Needs Review, Decided By, Priority, Category, Confidence, Reason, Suggested Action, Rule Flags, AI Status |

**n8n Data Tables (03 only)**. Create every column as type *String*:
- `ingest_request_log`: request_id, received_at, status, http_status, detail
- `ingest_dead_letters`: request_id, received_at, payload_json, last_error, attempts, status, resolved_at

**Typeform question refs (04):** `full_name`, `email`, `company`, `inquiry_type`, `team_size`, `budget`, `details`, plus optional hidden field `utm_source`. The Airtable fields are listed in 04's overview sticky; the upsert key is `Response ID`.

## 5. Error-handling design

### Principles applied in every workflow
1. **Every webhook path ends in a Respond node with an explicit status code.** n8n defaults to 200 even on error branches, so each one sets its code deliberately: 4xx = the caller's fault (decided before any work), 5xx = ours. Error bodies are `{ error, message, details? }` and never leak stack traces.
2. **Error outputs are always two-step:** `On Error = Continue (using error output)` **and** the red output wired to a real handler.
3. **Validate data, not just nodes.** A JavaScript error inside `{{ }}` silently becomes `null`. Every validator result is therefore guarded: a null result routes to a 500 plus an escalation.
4. **Degrade, don't drop.** When a store fails, the record still reaches a person (WhatsApp, Slack) or a durable queue (Data Table), and you get an email with the payload.
5. **AI fails safe.** 02 stores AI failures as `AI Status = FAILED`. 05 forwards any record the AI could not judge to a human (`decided_by = fail_safe`).

### n8n retry behaviour (verified in the n8n source)
- **`Retry On Fail` is skipped for Google Sheets, WhatsApp and Data Table nodes when an error output is enabled.** These nodes catch the error internally. Where a retry *and* an error branch are both needed (01, 03), the workflow uses an explicit loop: `error → Count Attempt → Attempts Left? → Wait (2 s, 4 s) → retry`.
- **Slack, Airtable, Gmail and the AI nodes do retry with an error output enabled**, but n8n re-runs the **whole node** if the **first item** failed. On multi-item nodes that would re-send items that already succeeded, so 05 retries Slack posts per failed item with a loop.
- **Google Sheets with an error output emits only the first input item on failure.** Batch writes (02, 05, the replay in 03) therefore keep the default *stop* behaviour with `Retry On Fail`. A final failure stops the run, sends an alert, and can be resumed with *Executions → Retry → From failed node*.
- **AI retries happen inside the model node** (`Max Retries = 3`, 60 s timeout), never as node-level retries on the chain or agent.

### Failure matrix

| WF | Failure | What happens | Caller sees | Email alert |
|---|---|---|---|---|
| 01 | Spam (honeypot) | Ignored | 200 | — |
| 01 | Invalid fields | Rejected | 400 + `details` | — |
| 01 | Validator crashed | Stopped | 500 | CRITICAL |
| 01 | Sheets down (3 attempts) | WhatsApp still sent, marked "NOT saved to CRM" | 202 | CRITICAL (full lead) |
| 01 | A WhatsApp number fails (3 attempts) | Only failed numbers retried | — | CRITICAL |
| 02 | Bad batch | Nothing processed | 400 + `details` | — |
| 02 | AI fails for some records | Saved with `AI Status = FAILED` | 202 | WARNING |
| 02 | Sheets down (3 retries) | Run stops, resumable from the failed node | 202 | ERROR |
| 03 | Invalid payload | Logged `REJECTED` | 400 + `details` | — |
| 03 | Sheets down (3 attempts) | Dead-letter queue, replayed every 15 min | 202 `queued` | WARNING |
| 03 | Sheets **and** DLQ down | Nothing stored | 503 + `Retry-After: 60` | CRITICAL (payload) |
| 04 | Answers unmappable (renamed refs) | Not written | — | CRITICAL |
| 04 | Airtable down (3 retries) | Slack still gets every answer | — | CRITICAL |
| 04 | Slack down (3 retries) | Record safe in Airtable | — | WARNING / CRITICAL |
| 05 | AI fails for a record | Forwarded to a human (`fail_safe`) | 202 | — |
| 05 | Review log (Sheets) down | Run stops before forwarding, resumable | 202 | ERROR |
| 05 | Slack post fails (3 attempts) | Decision kept in Review Log | — | CRITICAL |
| all | Email alert fails (3 retries) | Fallback channel (see §1) | — | fallback |

## 6. Before posting to the n8n community

The files already follow the template conventions below. They come from community sources rather than an official n8n page, so double-check the current rules in the n8n Creator Hub.
- One yellow overview sticky at the top left, 100–300 words, with **How it works** and **Setup steps**.
- White section stickies grouping the nodes.
- No credentials, API keys, pinned data or real contact details. Only placeholders such as `you@example.com`, `6281200000000` and `REPLACE_WITH_…`.

If you re-export a workflow after testing it on your own instance, check the export for your credential names, real email addresses, phone numbers or Slack IDs, and remove any pinned data before submitting. The safest option is to submit the files in this folder.

## 7. How these files were checked

The n8n-mcp server was unreachable during the build, so the files were **not** validated on a live n8n instance. Instead:
- Every node type, `typeVersion` and parameter name was checked against n8n's current source on GitHub.
- A structural checker covered: connections; error outputs wired both ways; Switch outputs; explicit status codes on Respond nodes; `$('Node')` references; exactly one Error Trigger; and the sticky-note conventions. It also syntax-checked all 385 JavaScript expressions with Node.js.
- 100 runtime tests evaluated the workflows' own expressions against sample payloads, including the error handler and its fallbacks in all five workflows.

Node versions used: Webhook 2, Set 3.4, IF 2.2, Switch 3.2, Google Sheets 4.5, Slack 2.3, Airtable 2.1, Gmail 2.1, Data Table 1.1, AI Agent 3.1, Basic LLM Chain 1.7, OpenAI Chat Model 1.2. They need a recent n8n (one that has Data Tables and AI Agent v3). Default models are `gpt-5-mini` (main) and `gpt-5` (JSON fixer).
