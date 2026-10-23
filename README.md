# n8n workflow templates: 5 standalone workflows with built-in error handling

Five production-style n8n workflows, each **fully self-contained**: no shared error workflow and no cross-workflow setup. Every file carries its own documentation on the canvas (one yellow overview sticky, white section stickies, a note on every node) and its own **built-in error handler**.

| File | Workflow name | Trigger | Services |
|------|---------------|---------|----------|
| `workflows/01-lead-capture-crm-whatsapp.json` | Capture website leads to a Google Sheets CRM with instant WhatsApp alerts | Webhook `POST /lead-capture` | Google Sheets, WhatsApp Cloud API, Gmail |
| `workflows/02-ai-data-processing-pipeline.json` | Triage customer feedback with OpenAI, Google Sheets and Gmail | Webhook `POST /feedback-triage` + Schedule (daily 08:00) | OpenAI, Google Sheets, Gmail |
| `workflows/03-webhook-to-sheets-reliable-ingest.json` | Log order webhooks to Google Sheets with a dead-letter queue and replay | Webhook `POST /ingest` + `GET /ingest-status` + Schedules (15 min replay, daily 07:00 report) | Google Sheets, n8n Data Tables, Gmail |
| `workflows/04-typeform-airtable-slack.json` | Route Typeform leads to Airtable and Slack with SLA follow-up reminders | Typeform Trigger + Schedule (15 min) | Typeform, Airtable, Slack, Gmail |
| `workflows/05-ai-agent-record-filter.json` | Pre-screen refund requests with an AI agent and approve risky ones in Slack | Webhook `POST /refund-requests` + one approval run per request (the workflow calls itself) | OpenAI (AI Agent + tools), Google Sheets, Slack, Gmail |

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
| 02 | Row in the **`Error Log`** sheet tab | create the empty tab |
| 03 | `INCIDENT` row in the existing **`ingest_request_log`** Data Table | none |
| 04, 05 | Message to the **Slack** ops channel in *Alert Settings* | invite the bot |

Each fallback reuses a credential the workflow already has, so the only extra credential is Gmail. To use Outlook or SMTP instead, swap out the **Email Alert** node.

**What reaches the handler.** Unexpected crashes, exhausted retries on nodes without an error branch, and deliberate escalations from the *Escalate ·* (Stop and Error) nodes. Escalation messages start with a tag that becomes the email's severity: `[CRITICAL]` = act now, `[WARNING]` = recovered automatically, no tag = `ERROR`.

**Testing the handler.** Open the workflow and execute **On Workflow Error** on its own. In manual mode it emits sample data, which proves the email and the fallback end to end.

## 2. Install one workflow (≈ 10–15 minutes each)

1. *Workflows → Import from File* → choose the JSON.
2. Open the yellow overview sticky (top left) and follow its **Setup** section.
3. Search the workflow for `REPLACE_WITH_` (spreadsheet IDs, WhatsApp phone-number ID, Airtable base/table IDs, Typeform form ID). The **Config** / **… Settings** nodes hold all business settings.
4. Leave *Settings → Error workflow* **empty**, so the built-in handler is used.
5. Run the `curl` test (01: first section sticky; 02, 03, 05: the notes of the webhook node), then **activate**.

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

**Google Sheets** (put these headers in row 1, except where a tab is marked "leave empty"):

| Workflow | Tab | Columns |
|---|---|---|
| 01 | `Leads` | Lead ID, Submitted At, Name, Email, Phone, Company, Message, Source, UTM Source, UTM Campaign, Status, Alert Status, Alerted At |
| 02 | `Feedback` | *(leave empty: the 22 headers are written on the first run; the upsert key is `Record Key`)* |
| 02 | `Error Log` | *(leave empty: headers are written on first use)* |
| 03 | `Events` | *(leave empty: the 9 headers are written on the first request; the upsert key is `Request ID`)* |
| 05 | `Review Log` | *(start with a new, empty tab: the 19 headers are written on the first run, and approvals update rows by `Review ID` relying on that exact column order)* |

**n8n Data Tables (03 only)**. Create every column as type *String*:
- `ingest_request_log`: request_id, received_at, status, http_status, detail
- `ingest_dead_letters`: request_id, received_at, payload_json, last_error, attempts, status, resolved_at

n8n finds a Data Table by name with a case-insensitive *contains* match and takes the most recently updated hit, so don't create other tables whose names contain these two (for example `ingest_request_log_old`).

**Typeform question refs (04):** `full_name`, `email`, `company`, `inquiry_type`, `team_size`, `budget`, `details`, plus optional hidden field `utm_source`.

**Airtable fields (04):** Response ID, Submitted At, Full Name, Email, Company, Inquiry Type, Team Size, Budget, Details, Routed To, Priority, Status (single select that includes `New`), UTM Source, Owner Slack ID, Slack Channel, Reminders Sent (number). The upsert key is `Response ID`. Your team moves Status away from `New` once a lead is handled; that stops the SLA reminders.

## 5. Error-handling design

### Principles applied in every workflow
1. **Every webhook path ends in a Respond node with an explicit status code.** n8n defaults to 200 even on error branches, so each one sets its code deliberately: 4xx = the caller's fault (decided before any work), 5xx = ours. Error bodies are `{ error, message, details? }` and never leak stack traces.
2. **Error outputs are always two-step:** `On Error = Continue (using error output)` **and** the red output wired to a real handler.
3. **Validate data, not just nodes.** A JavaScript error inside `{{ }}` silently becomes `null`. Every validator result is therefore guarded: a null result routes to a 500 plus an escalation.
4. **Degrade, don't drop.** When a store fails, the record still reaches a person (WhatsApp, Slack) or a durable queue (Data Table), and you get an email with the payload.
5. **AI fails safe.** 02 stores AI failures as `AI Status = FAILED`. 05 sends any request the AI could not judge to a person (`decided_by = fail_safe`).
6. **Personal data stays out of the model (02).** Emails, phone numbers, card numbers and IBANs are masked in code before any OpenAI call, and the raw text is dropped from the items.
7. **User text is inert everywhere it lands.** Sheets writes use cell format `RAW` (02, 03, 05), so text that starts with `=` is stored, never run as a formula. Text sent to Slack is escaped (04) or has `<` `>` replaced by `‹` `›` (05's approval message), so `<!channel>` in a form or request cannot ping anyone. HTML emails escape every value.

### n8n behaviour these designs rely on (verified in the source and live on n8n 2.18.5)
- **Retry loops must count attempts with `$runIndex`, not with a field on the item.** When a Google Sheets node fails while looking up the spreadsheet or tab (outage, wrong ID, renamed tab), its error item carries only `{ error }`. An item counter such as `_attempt` then resets on every pass and the loop never ends: the webhook hangs and the API is called hundreds of times. 03 counts with `$runIndex` and rebuilds the row on every attempt. **01 still uses item counters (`_crm_attempt`, `_wa_attempt`) and needs the same fix.**
- **"Continue (using error output)" only routes to the error output when the node catches the error itself** (Gmail, Airtable, Slack messages, Data Tables, most Sheets operations). If the node throws, n8n passes the *input* items on the **success** output. Slack's send-and-wait does this when Slack refuses the message, so 05 checks what comes out of it.
- **`Retry On Fail` re-runs the whole node when the first output item has a top-level `error` field**, even if the node succeeded. A node that passes its input through (a Sheets append in auto-map mode fed by an error output) then writes the same row on every try. 02's **Error Log Row** renames the field first. With several items, a retry also re-sends items that already went out, so 04's SLA reminders use no node retry: a reminder that failed is simply due again in the next run.
- **Slack error items carry no link to their input item.** 04 works out which reminders failed from the ones it could mark as sent.
- **Send-and-wait returns the request unchanged on a timeout** (and outputs nothing at all without *Always Output Data*). 05 tells a timeout from a refused message by the time elapsed.
- **The send-and-wait message decodes HTML entities before posting**, so `&lt;` turns back into `<`. 05 replaces `<` `>` in request text with `‹` `›` instead.
- **Google Sheets "Map each column" on an empty tab silently auto-maps the input item instead**, writing headers like `request_id` or `validation`. 02, 03 and 05 build an item that *is* the row and auto-map it, so an empty tab gets the right headers. Sheets 4.4+ updates in "Map each column" mode also compare the full header row with the node's stored schema, so 05's **Record Human Decision** lists every column in order.
- **A Typeform Trigger needs a `webhookId` in the JSON.** Without one, n8n registers an encoded path that never matches the incoming request (404), so no submission arrives. 04 now ships with one.
- **Google Sheets with an error output emits only the first input item on failure.** Batch writes (02, 05, the replay in 03) therefore keep the default *stop* behaviour with `Retry On Fail`. A final failure stops the run, sends an alert, and can be resumed with *Executions → Retry → From failed node*; the saved AI results are reused.
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
| 02 | AI fails for some records | Saved with `AI Status = FAILED`; re-send them to re-process | 202 | WARNING |
| 02 | Sheets down (3 retries) | Run stops, resumable from the failed node (AI results reused) | 202 | ERROR |
| 02 | Urgent-feedback email fails (3 retries) | Rows already saved; the incident lists the items | 202 | CRITICAL |
| 02 | Digest AI fails | Digest still sent, numbers only | — | — |
| 02 | Digest read or send fails (3 retries) | Run stops | — | ERROR |
| 03 | Invalid payload | Logged `REJECTED` | 400 + `details` | — |
| 03 | Sheets down (3 attempts) | Dead-letter queue, replayed every 15 min | 202 `queued` | WARNING |
| 03 | Sheets **and** DLQ down | Nothing stored | 503 + `Retry-After: 60` | CRITICAL (payload) |
| 03 | Replay still failing | Rows stay `PENDING`; the daily report flags stuck ones | — | ERROR (each run) |
| 03 | Status lookup for an unknown ID | — | 404 | — |
| 04 | Answers unmappable (renamed refs) | Not written | — | CRITICAL |
| 04 | Returning-contact lookup fails | Lead processed without history | — | — |
| 04 | Airtable down (3 retries) | Slack still gets every answer | — | CRITICAL |
| 04 | Slack down (3 retries) | Record safe in Airtable | — | WARNING / CRITICAL |
| 04 | SLA reminder not posted | Not counted; due again next run | — | WARNING |
| 05 | AI fails for a request | Sent to a person (`fail_safe`) | 202 | — |
| 05 | Review log (Sheets) down | Run stops before any approval, resumable | 202 | ERROR |
| 05 | Slack refuses the approval message (3 attempts) | Request stays `pending` in the log | — | CRITICAL (request details) |
| 05 | Nobody answers within 48 h | Logged as `expired` | — | — |
| all | Email alert fails (3 retries) | Fallback channel (see §1) | — | fallback |

## 6. Before posting to the n8n community

Rules from n8n's *Template submission guidelines* and *Sticky note guidelines for templates* (Creator Hub pages on n8n.notion.site, checked 2026-10-21 and 2026-10-24):
- Low-effort templates are not published; a template should show "thoughtful design, utility, and real-world value". This is why the first version of 02 was rejected as "too basic"; 03–05 were then given concrete use cases too.
- Title: sentence case, action verb + what + where, name the main apps, no emojis or hype (e.g. "Sync contacts from Pipedrive to HubSpot").
- Exactly one yellow overview sticky at the top left, 100–300 words, with **How it works** and **Setup** (optional **Customization**).
- White section stickies for workflows with 4+ nodes, **under 50 words each**: a `##` title of 3–6 words in sentence case plus at most one short line. Each one wraps a small cluster of 2–4 nodes tightly, without large empty areas. Put longer explanations in node notes.
- n8n's official [Auto-generate sticky notes](https://n8n.io/workflows/13868-auto-generate-sticky-notes-and-rename-nodes/) template shows the expected result: overview with a numbered **How it works** (one sentence per step), **Setup steps** as `- [ ]` checkboxes, optional **Customization**; about one section sticky per 3 nodes.
- No credentials, API keys, pinned data or real contact details. Only placeholders such as `you@example.com`, `6281200000000` and `REPLACE_WITH_…`.
- Description: about 200 words, Markdown, no HTML.

On 2026-10-24 n8n returned 03 because its stickies did not meet their quality standards: six large section stickies with 25–40 words of implementation detail, and a dense overview with a `curl` block. 02–05 now follow the official template's layout: 9–13 tight section stickies each and a 200–236-word overview, with the `curl` tests moved to the webhook node notes. **01 still has the old style** (section stickies of 86–161 words); restyle it before submitting. Ready-to-paste titles and descriptions for 02–05 are in [`TEMPLATE-SUBMISSIONS.md`](TEMPLATE-SUBMISSIONS.md).

If you re-export a workflow after testing it on your own instance, check the export for your credential names, real email addresses, phone numbers or Slack IDs, and remove any pinned data before submitting. The safest option is to submit the files in this folder.

## 7. How these files were checked

**01** (built 2026-10-19) was **not** validated on a live n8n instance: its node types, versions and parameters were checked against n8n's source, a structural checker covered wiring and expressions, and runtime tests evaluated its expressions against sample payloads. The live tests of 03 show that its item-counter retry loops are unsafe (see §5).

**02–05** (rebuilt 2026-10-21) ran end to end on a real **n8n 2.18.5** instance: an isolated local install where OpenAI, Google Sheets, Gmail, Typeform, Airtable and Slack were replaced by a local mock API. The Airtable mock evaluates the workflows' `filterByFormula` strings with a small formula interpreter, and the Typeform mock signs payloads like Typeform does.
- **02 — 48 checks:** webhook answers (400, 403, 202 before the AI step); masking (no raw email, phone, card or IBAN reached the model); schema-checked AI output with auto-fix and fail-safe rows; upsert without duplicates and `RAW` writes; urgent email; error handler with Gmail down and with Sheets down; *Retry → From failed node* reusing AI results; digest with AI brief, AI failure, repaired JSON, quiet day and empty sheet; the real 08:00 Schedule Trigger fired.
- **03 — 33 checks:** stored, resent (no duplicate) and rejected requests; Sheets down → retried twice, queued, `202`, WARNING; status endpoint (200, 404, 400, 403); replay writes the row and resolves the dead letter; Sheets **and** queue down → `503` + `Retry-After`; daily report numbers; email down → exactly one `INCIDENT` row.
- **04 — 24 checks:** Typeform webhook registration and signature check (wrong signature → 401); routing, HOT flag, stored owner and channel; returning-contact history; `<!channel>` in an answer escaped; unmappable answers, Airtable down and Slack down escalations; SLA reminders (only due leads, reminder 2 tags the manager, counters updated, nothing due on the next run); partial Slack failure counted correctly; Slack ops fallback.
- **05 — 22 checks:** hard rules, AI, low-confidence and fail-safe decisions; the agent calls the history tool and receives an earlier declined refund; one waiting approval run per request with signed Approve / Decline links; clicks written back; a link-preview bot does not approve; unanswered approvals marked `expired`; `<!channel>` neutralised; Slack refusing the message → CRITICAL with the request.

These live tests found bugs that the original 2026-10-19 versions also had: the endless Sheets retry loop (03), the missing Typeform `webhookId` (04, no submission ever arrived), the Slack retry loop that could not count attempts (old 05), headers created from the wrong fields on an empty tab (03), and duplicate fallback rows (02).

No real OpenAI, Google, Typeform, Airtable or Slack account was used, so check answer quality, quotas and app permissions on your own instance before activating.

Node versions used: Webhook 2, Schedule Trigger 1.2, Typeform Trigger 1.1, Execute Workflow 1.2, Execute Workflow Trigger 1.1, Set 3.4, IF 2.2, Switch 3.2, Google Sheets 4.5, Google Sheets Tool 4.5, Slack 2.3, Airtable 2.1, Gmail 2.1, Data Table 1.1, AI Agent 3.1, Think Tool 1, Basic LLM Chain 1.7, Structured Output Parser 1.3, OpenAI Chat Model 1.2. They need a recent n8n (one that has Data Tables and AI Agent v3). Default models are `gpt-5-mini` (main) and `gpt-5` (JSON fixer).
