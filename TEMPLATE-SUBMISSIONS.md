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

---

## 06 · `workflows/06-whatsapp-ai-sales-agent.json`

**Title:** Answer WhatsApp sales chats with an AI agent, a Google Sheets catalog and lead capture

```markdown
## Who it's for
Online shops and small businesses that sell through WhatsApp and lose customers when nobody answers at night or on busy days.

## What it does
- Replies to WhatsApp messages 24/7 with an OpenAI agent that searches your Google Sheets product catalog and only quotes real prices and stock.
- Groups messages sent in quick succession into one answer, skips repeated webhooks and transcribes voice notes.
- Keeps one lead row per customer in Google Sheets (stage, interest, budget) and logs confirmed orders in an Orders tab, with an email to your team.
- Hands complaints and "talk to a person" requests to your team, who reply from a protected n8n form while the bot stays quiet.
- Sends one follow-up to quiet warm leads before the 24-hour WhatsApp window closes.
- Built-in error handler: failures are emailed, with a Google Sheets log as fallback.

## How to set up
1. Add WhatsApp (trigger and API), OpenAI, Google Sheets, Gmail and Basic Auth credentials.
2. Create the two Data Tables and the spreadsheet tabs listed in the overview note.
3. Edit Sales Settings, then activate and point your WhatsApp app to the trigger.

## How to customize
Change the selling rules and tone in the Sales Agent system message, and the follow-up timing in Sales Settings.
```

---

## 07 · `workflows/07-ai-lead-generation-outreach.json`

**Title:** Find Google Maps leads with SerpAPI, score them with OpenAI and send personal emails with Gmail

```markdown
## Who it's for
Agencies, freelancers and B2B sellers who prospect local businesses and want a steady flow of qualified leads with personal first emails.

## What it does
- Every morning, runs your "keyword | location" searches on Google Maps through SerpAPI and skips businesses already in your sheet.
- Reads each website for email addresses, Instagram and what the business does. An email is only used when the site actually shows it.
- Scores every lead against your ideal customer with OpenAI and writes a personal opening line, using a JSON schema with automatic repair.
- Saves leads to Google Sheets and emails you a daily summary.
- On weekdays, sends a few emails per hour from Gmail: first emails to the best new leads, then follow-ups in the same thread.
- Stops the sequence when a lead replies and marks "stop" replies as opted out.
- Built-in error handler with a Google Sheets fallback.

## How to set up
1. Add SerpAPI, OpenAI, Google Sheets and Gmail credentials.
2. Create a spreadsheet with empty Leads and Error Log tabs.
3. Edit Prospecting Settings and Outreach Settings, set the workflow time zone and activate.

## How to customize
Change the scoring rules in Score Lead and the email style in Write Email.
```

---

## 08 · `workflows/08-content-repurposing-engine.json`

**Title:** Turn YouTube videos and podcasts into 30 posts with OpenAI, Google Docs and a Sheets content calendar

```markdown
## Who it's for
Creators, coaches and marketing teams who publish long videos or podcasts and want a month of posts from each one.

## What it does
- Starts from a form link (YouTube, TikTok or podcast) or, optionally, from every new upload on your YouTube channel.
- Gets the transcript through Supadata and waits for long videos to finish processing.
- Has OpenAI write a content brief first, then an SEO blog article and 30 social pieces: LinkedIn posts, an X thread and posts, Instagram captions, Facebook posts, a newsletter, short-video scripts and quote cards.
- Saves the article as a Google Doc and gives every piece a row and a publish slot in a Google Sheets content calendar, then emails you the pack.
- Publishes approved LinkedIn posts at their slot and sends a morning reminder for the other platforms.
- Built-in error handler with a Google Sheets fallback.

## How to set up
1. Add Supadata (Header Auth), OpenAI, Google Drive, Google Sheets, Gmail and LinkedIn credentials.
2. Create a spreadsheet with empty Content Calendar and Error Log tabs.
3. Edit the two settings nodes and activate. Enable the YouTube feed trigger if you want automatic packs.

## How to customize
Change the publish cadence in Build Calendar Rows and the voice in the writing prompts.
```

---

## 09 · `workflows/09-ai-bookkeeping-invoices.json`

**Title:** Read invoices and receipts with OpenAI, track them in Google Sheets and send payment reminders with Gmail

```markdown
## Who it's for
Small businesses and freelancers who keep their books in a spreadsheet and spend hours typing in receipts and chasing late payers.

## What it does
- Collects invoices and receipts from labelled Gmail attachments or a protected upload form (PDFs and photos).
- Reads each document with OpenAI into structured data: party, dates, totals, tax, category and payment status.
- Checks that subtotal plus tax equals the total, that dates are plausible and that the AI is confident. Doubtful documents are saved as "Needs review", and duplicates are skipped.
- Archives every file in Google Drive and records it in a Transactions tab, then emails you a summary.
- Adds your own unpaid sales invoices to an Invoices tab and sends clients up to three polite reminders after the due date.
- Every Monday, emails a cash-flow report comparing last week with the week before, with bills due, overdue receivables and a short AI note.
- Built-in error handler with a Google Sheets fallback.

## How to set up
1. Add Gmail, OpenAI, Google Drive, Google Sheets and Basic Auth credentials.
2. Create the spreadsheet tabs and the Gmail label from the overview note.
3. Pick a Drive folder and edit the three settings nodes.

## How to customize
Change the categories in Finance Settings and the reminder texts in Reminder Settings.
```

---

## 10 · `workflows/10-rag-support-agent.json`

**Title:** Answer customer questions from your own docs with an OpenAI agent, Pinecone and automatic support tickets

```markdown
## Who it's for
Support teams and online businesses whose customers ask the same questions every day, and whose answers already live in SOPs and FAQ documents.

## What it does
- Offers a hosted chat page and embeddable widget where an OpenAI agent answers only from your documents in Pinecone and streams the reply.
- Logs every question it cannot answer as a knowledge gap, instead of guessing.
- Turns problems that need a person (refunds, damaged items, account issues) into a ticket in Google Sheets and an email to your team, after asking for the customer's name and email.
- Checks your Google Drive docs folder every hour and rebuilds the knowledge base after any change.
- Every Monday, emails insights: top topics, complaints and the documentation updates that would close the gaps.
- Built-in error handler with a Google Sheets fallback.

## How to set up
1. Add OpenAI, Pinecone, Google Drive, Google Sheets and Gmail credentials.
2. Create the Pinecone index, the Data Table and the spreadsheet tabs from the overview note.
3. Put your docs in a Drive folder, set its ID, edit the settings nodes and activate.

## How to customize
Adjust tone and ticket rules in the Support Agent system message.
```

---

## 11 · `workflows/11-multi-store-order-hub.json`

**Title:** Sync orders and stock across Shopify, WooCommerce and marketplaces with Google Sheets and WhatsApp reports

```markdown
## Who it's for
Online sellers who sell the same products on Shopify, WooCommerce and marketplaces such as Shopee or Tokopedia, and keep overselling because stock is updated by hand.

## What it does
- Receives new orders from Shopify, WooCommerce and a secured webhook for marketplaces, POS or aggregators.
- Normalizes every order into one format, flags orders without SKUs and skips repeated webhooks.
- Logs each order once in a Google Sheets Orders tab and writes one stock movement per order line to an append-only ledger, so two orders at the same moment never overwrite each other.
- Calculates current stock from your last stock count plus all movements and pushes it back to Shopify and WooCommerce.
- Sends an email and a WhatsApp message when a SKU drops to its alert level.
- Every evening, sends a sales report by email and a WhatsApp summary: revenue, orders, best sellers and low stock.
- Built-in error handler with a Google Sheets fallback.

## How to set up
1. Add Shopify, WooCommerce, Header Auth, Google Sheets, Gmail and WhatsApp credentials.
2. Create the Data Table and the Inventory, Orders and Error Log tabs from the overview note.
3. Get the two WhatsApp templates approved and edit the two settings nodes.

## How to customize
After a stock count, update Start Stock and Counted At in the Inventory tab.
```

---

## 12 · `workflows/12-ai-cv-screening-interviews.json`

**Title:** Screen CVs with OpenAI, rank candidates in Google Sheets and book interviews with Google Calendar

```markdown
## Who it's for
Recruiters, HR teams and founders who receive many applications per role and want a fair first screening and less back-and-forth about interview times.

## What it does
- Collects applications through an n8n form with a PDF CV and skips repeat applications.
- Screens blind: OpenAI first turns the CV into a profile without name, age, gender, photo, schools or employers, and a second call scores only that profile against the role's must-haves from your Jobs tab.
- Saves every candidate in Google Sheets with score, met and missing must-haves, strengths, gaps and three interview questions, and stores the CV in Google Drive.
- Emails strong matches a link to pick a free time from your Google Calendar, re-checks the time before booking, and sends a Google Meet invite plus a briefing for the interviewer.
- Sends HR a ranked digest every morning. HR sets Status to Invite or Rejected in the sheet, and the next run sends the invitation or a polite rejection. No rejection goes out without HR.
- Built-in error handler with a Google Sheets fallback.

## How to set up
1. Add OpenAI, Google Sheets, Google Drive, Google Calendar and Gmail credentials.
2. Create the Jobs, Candidates and Error Log tabs from the overview note.
3. Match the form's Position options to your job titles and edit Hiring Settings.

## How to customize
Change score thresholds, working hours, interview length and email texts in Hiring Settings.
```

---

## 13 · `workflows/13-client-onboarding.json`

**Title:** Onboard paying clients with Stripe, Google Drive, a Docs contract, Trello and Gmail

```markdown
## Who it's for
Agencies, studios and consultants who repeat the same onboarding steps for every new client and want them done the moment the payment arrives.

## What it does
- Starts when a Stripe Checkout or Payment Link is paid, or when your team enters a bank transfer in an internal form. Each payment onboards once, even when Stripe resends the event.
- Creates a client folder with your standard subfolders in Google Drive.
- Copies your Google Docs contract template and fills in client, package, amount and dates.
- Shares the contract for comments and an assets folder where the client can upload files.
- Creates a Trello board from your template board and logs the client in Google Sheets.
- Sends the client a welcome email with the contract, a brief form, the shared folder and your kick-off booking link, and gives your team all the links.
- Saves the client's brief as a Google Doc in their folder and reminds clients who have not sent it, up to two times.
- Built-in error handler with a Google Sheets fallback.

## How to set up
1. Add Stripe, Google Drive, Google Docs, Google Sheets, Trello, Gmail and Basic Auth credentials.
2. Prepare the contract template with placeholders and a Trello template board.
3. Create the Clients and Error Log tabs and edit Onboarding Settings.

## How to customize
Change subfolders, packages and reminder timing in Onboarding Settings.
```
