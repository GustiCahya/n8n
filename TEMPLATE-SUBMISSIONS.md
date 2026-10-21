# Template submission texts

Copy the title and the description into the n8n Creator Portal for each workflow. The descriptions follow n8n's guidelines: about 200 words, Markdown, no HTML.

---

## 02 · `workflows/02-ai-data-processing-pipeline.json`

**Title:** Triage customer feedback with OpenAI, Google Sheets and Gmail

```markdown
## Who it's for
Support, product and customer-experience teams that collect free-text feedback (helpdesk tickets, app reviews, NPS comments, contact forms) in several tools and want every message triaged without reading each one.

## What it does
- Receives batches of messages on one header-protected webhook and answers `202` immediately.
- Masks emails, phone numbers, card numbers and IBANs in code **before** anything reaches OpenAI.
- Classifies each message (category, sentiment, priority, churn risk) and writes a summary and a suggested next step, validated against a JSON Schema.
- Upserts rows into Google Sheets by record key, so re-sent messages update their row instead of duplicating it.
- Sends one Gmail alert per batch for urgent, high-priority or churn-risk items.
- Every morning, compares the last 24 hours with the 24 before, flags category spikes and emails an AI-written brief with themes and recommended actions.
- Built-in error handler: failures are emailed, with a Google Sheets log as fallback.

## How to set up
1. Add OpenAI, Google Sheets, Gmail and Header Auth credentials.
2. Create a spreadsheet with empty `Feedback` and `Error Log` tabs and select it in the three Google Sheets nodes.
3. Set recipients in the three Settings nodes, activate, and send the test request from the overview note.

## How to customize
Edit the categories in the output schema and system prompt together. Alert threshold, summary language and batch limits are in Triage Settings.
```

---

## 03 · `workflows/03-webhook-to-sheets-reliable-ingest.json`

**Title:** Log order webhooks to Google Sheets with a dead-letter queue and replay

```markdown
## Who it's for
Teams that receive order, payment or sign-up webhooks and keep them in Google Sheets, but cannot afford to lose a request when Google has an outage or a sender retries.

## What it does
- Validates each payload and assigns a request ID (from `X-Request-ID` or generated). Bad payloads get `400` with per-field details.
- Upserts the row by request ID, so resends never create duplicates. Values are written as plain text, so something like `=HYPERLINK(...)` never runs as a formula.
- Retries a failed write in-request (2 s, then 4 s). If Sheets is still down, the request is parked in an n8n Data Table, the caller gets `202`, and the queue is replayed every 15 minutes. If even the queue fails, the caller gets `503` with `Retry-After`.
- Lets callers check any request with `GET /webhook/ingest-status?request_id=…`.
- Emails a daily health report: stored, queued and rejected counts, the most common validation errors and requests stuck in the queue.
- Built-in error handler: every degraded path is emailed, with an incident row as fallback.

## How to set up
1. Add Header Auth, Google Sheets and Gmail credentials.
2. Create an empty `Events` tab and the two Data Tables listed in the overview note.
3. Select the spreadsheet in both Sheets nodes and set your emails.

## How to customize
Change the required fields and their checks in **Build Request Envelope**.
```

---

## 04 · `workflows/04-typeform-airtable-slack.json`

**Title:** Route Typeform leads to Airtable and Slack with SLA follow-up reminders

```markdown
## Who it's for
Sales, support and partnership teams that collect inquiries with Typeform and want every lead in Airtable, the right owner pinged in Slack, and no lead left unanswered.

## What it does
- Maps answers by question reference, so renaming a question never breaks the flow.
- Routes each inquiry to Sales, Support, Partnership or General with keyword rules, and flags large sales leads as HOT.
- Looks up earlier submissions from the same email, so the owner sees returning contacts.
- Upserts the lead in Airtable by Response ID, together with its owner and Slack channel.
- Posts a Block Kit message that @-mentions the owner, with the answers, the history and a button to the record. Form text is escaped, so it cannot ping a whole channel.
- Every 15 minutes, finds leads still marked New after their SLA (HOT 30 minutes, others 4 hours), reminds the owner, and tags a manager on the second reminder.
- If Airtable or Slack fails, the lead still reaches a person and you get an email. A built-in error handler covers everything else.

## How to set up
1. Set the Typeform question refs and create the Airtable fields listed in the overview note.
2. Add Typeform, Airtable, Slack and Gmail credentials, then select your base and table.
3. Edit the routing table, the SLA settings and the alert email.

## How to customize
Adjust the keyword rules in **Pick Route & Priority** or the SLA minutes in **SLA Settings**.
```

---

## 05 · `workflows/05-ai-agent-record-filter.json`

**Title:** Pre-screen refund requests with an AI agent and approve risky ones in Slack

```markdown
## Who it's for
E-commerce and SaaS teams that handle refund, cancellation and payment-dispute requests and want routine ones approved fast while risky ones always get a human decision.

## What it does
- Accepts batches of requests on a secured webhook and answers `202` immediately.
- Applies hard rules in code first: risk keywords (chargeback, lawyer, fraud…) or large amounts always go to a person.
- An AI agent (OpenAI) reviews each request, can look up the customer's earlier decisions in Google Sheets, and returns schema-validated JSON. If it fails, the request goes to a person.
- Logs every decision to a `Review Log` sheet.
- Starts one approval run per risky request: a Slack message with Approve / Decline buttons that waits up to 48 hours. The answer, or `expired`, is written back to the log, where the agent sees it next time.
- Neutralises request text in Slack, ignores link-preview bots, and escalates by email if Slack refuses the message. A built-in error handler covers the rest.

## How to set up
1. Add Header Auth, OpenAI, Google Sheets, Slack and Gmail credentials.
2. Create an empty `Review Log` tab and select the spreadsheet in the three Sheets nodes.
3. Edit **Config** (approval channel, thresholds) and invite the Slack bot.

## How to customize
Connect your payment provider after **Record Human Decision** to issue approved refunds automatically.
```
