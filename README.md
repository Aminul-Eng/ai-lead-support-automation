# AI-Powered Lead & Customer Support Automation System

> An end-to-end AI automation system built with n8n for intelligent lead qualification, CRM automation, automated follow-up, RAG-based customer support, deterministic safety controls, human handoff, idempotency protection, and operational observability.

![n8n](https://img.shields.io/badge/Workflow-n8n-orange)
![AI](https://img.shields.io/badge/AI-Groq-blue)
![Embeddings](https://img.shields.io/badge/Embeddings-Google%20Gemini-blue)
![Vector DB](https://img.shields.io/badge/Vector%20Database-Pinecone-green)
![CRM](https://img.shields.io/badge/CRM-HighLevel-purple)
![Status](https://img.shields.io/badge/DEV%20E2E-Tested-success)

---

## Project Status

**Development build — inactive / unpublished**

This repository contains a sanitized portfolio implementation of the workflow.

The system has been tested end-to-end in a controlled DEV environment using connected AI, CRM, email, vector database, and n8n Data Table integrations.

It is intended to be used as:

- A portfolio project
- A reusable automation architecture
- A client implementation baseline
- A reference for AI + CRM + RAG automation design

Client-specific configuration, credentials, business rules, knowledge-base data, and UAT are required before production deployment.

---

# Business Problem

Businesses often receive both **sales inquiries** and **customer support requests** through the same website, form, inbox, or webhook.

Without automation, someone must manually:

- Read every incoming message
- Determine whether it is a sales or support request
- Identify high-value leads
- Update the CRM
- Send follow-up emails
- Search support documentation
- Decide whether an AI-generated answer is reliable
- Escalate uncertain cases
- Track failures and duplicate requests

This creates several operational problems:

- Slow lead response time
- Inconsistent qualification
- Missed high-value opportunities
- Duplicate CRM actions
- Repeated customer emails
- Support bottlenecks
- AI hallucination risk
- Poor execution visibility
- Difficult troubleshooting

---

# Solution

This workflow provides a unified automation layer for both **lead management** and **customer support**.

Every incoming request is:

1. Validated
2. Normalized
3. Checked for duplicates
4. Classified by intent
5. Routed into the appropriate business pipeline

The two main paths are:

```text
SALES
→ AI Lead Qualification
→ Deterministic Scoring
→ CRM
→ Follow-Up
→ Sales Alert

SUPPORT
→ Gemini Embeddings
→ Pinecone Retrieval
→ Groq RAG Generation
→ Deterministic Grounding Gate
→ Auto Response OR Human Handoff
```

Operational events are logged throughout the workflow for request-level traceability.

---

# System Architecture

```text
                       ┌─────────────────────────┐
                       │ Website / Form / Webhook│
                       └────────────┬────────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │ Intake & Validation │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │ Duplicate Detection │
                         └──────────┬──────────┘
                                    │
                                    ▼
                      ┌──────────────────────────┐
                      │ AI Intent Classification │
                      └────────────┬─────────────┘
                                   │
                 ┌─────────────────┴─────────────────┐
                 │                                   │
                 ▼                                   ▼
              SALES                               SUPPORT
                 │                                   │
                 ▼                                   ▼
       AI Lead Qualification               Query Normalization
                 │                                   │
                 ▼                                   ▼
      Deterministic Lead Score               Gemini Embeddings
                 │                                   │
                 ▼                                   ▼
        HIGH / MEDIUM / LOW                 Pinecone Retrieval
                 │                                   │
                 ▼                                   ▼
          HighLevel CRM                    Context Validation
                 │                                   │
                 ▼                                   ▼
       Customer Follow-Up                 Groq RAG Generation
                 │                                   │
                 ▼                                   ▼
       HIGH Sales Alert              Deterministic Grounding Gate
                                                    │
                                         ┌──────────┴──────────┐
                                         │                     │
                                         ▼                     ▼
                                  Auto Response          Human Handoff
                                         │                     │
                                         └──────────┬──────────┘
                                                    │
                                                    ▼
                                         Operational Logging
```

---

# Full Workflow Overview

![Workflow Overview](screenshots/00-workflow-overview.png)

---

# Workflow Modules

The workflow is organized into modular business components for easier maintenance, debugging, testing, and client customization.

| Module | Name | Responsibility |
|---|---|---|
| 01 | Intake & Validation | Receives webhook payloads, normalizes data, validates required fields, and rejects invalid requests |
| 02 | Deduplication & Intent Classification | Prevents repeated processing and classifies requests as Sales, Support, General, or Unknown |
| 03 | AI-Assisted Lead Scoring | Extracts AI signals and combines them with deterministic business rules |
| 04 | CRM Integration | Searches, creates, or updates HighLevel contacts and manages HIGH-lead opportunities |
| 05 | Follow-Up & Sales Alerts | Sends lead-specific customer emails and HIGH-priority internal sales alerts |
| 06 | Retrieval-Augmented Support Generation | Uses Gemini embeddings, Pinecone retrieval, and Groq to generate knowledge-grounded answers |
| 07 | Deterministic Grounding Gate & Human Handoff | Allows automated answers only when safety requirements pass |
| 08 | Operational Logging & Error Handling | Records structured events and supports safe failure handling across the workflow |

---

# 01 — Intake & Validation

The workflow starts with a single webhook.

Incoming data is normalized into a consistent structure before any AI, CRM, or email operation occurs.

Typical fields include:

```json
{
  "name": "Customer Name",
  "email": "customer@example.com",
  "phone": "",
  "company": "Example Company",
  "job_title": "Operations Manager",
  "budget": "$5000",
  "message": "We need help automating our lead process.",
  "source": "website"
}
```

The validation layer checks required fields such as:

- Name
- Valid email format
- Message

Invalid requests are rejected safely before downstream processing.

![Intake Validation](screenshots/01-intake-validation.png)

---

# 02 — Deduplication & Intent Classification

Before processing a request, the workflow computes an idempotency key from normalized request data.

The intake deduplication mechanism is based on:

```text
normalized email + normalized message
```

If the same request has already been processed, the workflow returns a duplicate response instead of calling AI, CRM, or email services again.

Example:

```json
{
  "status": "duplicate",
  "message": "This request has already been received.",
  "duplicate": true
}
```

After deduplication, Groq classifies the request into one of four intents:

| Intent | Meaning |
|---|---|
| `sales` | Sales or automation-service opportunity |
| `support` | Existing customer support request |
| `general` | General information request |
| `unknown` | Low-confidence or uncertain classification |

Unknown or unsafe classifications can be routed toward review instead of forcing an automated business decision.

![Deduplication and Intent Classification](screenshots/02-deduplication-intent-classification.png)

---

# 03 — AI-Assisted Lead Scoring

Sales requests are passed through an AI-assisted qualification layer.

The scoring system intentionally combines:

1. **AI-extracted semantic signals**
2. **Deterministic business rules**

The AI does not directly control the final score.

---

## AI-Derived Signals

Groq analyzes the message and returns structured business signals.

| Signal | Points |
|---|---:|
| Clear business need | +2 |
| Specific automation requirement | +2 |
| Urgency or timeline | +1 |
| Pricing / demo / call request | +1 |
| Strong purchase intent | +1 |

Maximum AI-derived score:

```text
7
```

---

## Deterministic Signals

The workflow independently evaluates structured fields.

| Signal | Points |
|---|---:|
| Decision-maker job title | +1 |
| Budget provided | +1 |
| High budget threshold reached | +1 |

High-budget threshold used in the DEV implementation:

```text
>= $10,000
```

---

## Final Lead Score

```text
Maximum Score = 10
```

Lead categories:

| Score | Category |
|---:|---|
| 8–10 | HIGH |
| 5–7 | MEDIUM |
| 1–4 | LOW |

Malformed or unusable AI output is routed safely rather than producing an unreliable qualification result.

---

## Example HIGH Lead Result

```json
{
  "status": "qualified",
  "lead_score": 10,
  "lead_category": "high",
  "summary": "Approved budget for CRM automation and discovery call request.",
  "recommended_action": "Schedule discovery call and provide pricing details.",
  "crm_contact_id": "REDACTED",
  "crm_opportunity_id": "REDACTED",
  "crm_opportunity_status": "existing",
  "email_sent": true,
  "sales_alert_sent": true
}
```

---

# 04 — HighLevel CRM Integration

Qualified leads are synchronized with **HighLevel CRM**.

The CRM flow performs the following operations:

```text
Search Contact by Email
        │
        ▼
Resolve Existing Contact
        │
   ┌────┴────┐
   │         │
Existing    New
   │         │
Update     Create
   │         │
   └────┬────┘
        │
        ▼
Resolve Contact ID
        │
        ▼
Route by Lead Category
```

For HIGH leads, the workflow additionally performs:

```text
Check Existing Opportunity
        │
   ┌────┴─────┐
   │          │
 Exists     Missing
   │          │
 Reuse      Create
   │          │
   └────┬─────┘
        │
        ▼
Continue Follow-Up
```

This helps reduce unnecessary opportunity duplication.

---

## CRM Failure Handling

CRM API errors do not crash the workflow blindly.

CRM failures are routed through a dedicated failure handler that:

- Classifies the failure
- Sanitizes technical information
- Writes an operational log
- Returns a safe response
- Avoids exposing internal API details to the caller

---

# AI Lead Scoring + CRM Screenshot

![AI Lead Scoring and CRM](screenshots/03-ai-lead-scoring-crm.png)

---

# 05 — Automated Follow-Up & Sales Alerts

After qualification and CRM processing, leads receive category-specific follow-up emails.

---

## HIGH Leads

HIGH leads can trigger:

```text
Customer Follow-Up Email
        +
Internal Sales Alert
        +
CRM Opportunity Processing
```

---

## MEDIUM Leads

MEDIUM leads receive:

```text
Customer Follow-Up Email
```

No HIGH-priority internal sales alert is sent.

---

## LOW Leads

LOW leads receive an appropriate low-intent follow-up.

This allows the workflow to nurture lower-priority opportunities without treating them as hot leads.

---

## Email Protection

Customer email sends use check-before-send tracking.

Tracking keys include:

```text
request_id + email + lead_category
```

The tracking record is written only after a successful Gmail API send response.

This is designed to reduce duplicate customer-facing emails during retries or repeated processing.

It is **not presented as an absolute exactly-once guarantee** across external APIs.

![Follow-Up and Sales Alerts](screenshots/04-followup-sales-alerts.png)

---

# 06 — RAG-Based Customer Support

Support requests follow a Retrieval-Augmented Generation pipeline.

```text
Customer Question
       │
       ▼
Normalize Query
       │
       ▼
Google Gemini Embedding
       │
       ▼
Pinecone Vector Search
       │
       ▼
Validate Retrieved Context
       │
       ▼
Groq RAG Generation
       │
       ▼
Validate AI Answer
       │
       ▼
Deterministic Resolution Gate
```

---

## Embedding Configuration

| Setting | Value |
|---|---|
| Embedding model | `gemini-embedding-2` |
| Vector dimension | `3072` |
| Similarity metric | Cosine |
| Retrieval | Top 4 matches |

The development index uses a 3072-dimensional vector configuration compatible with the selected Gemini embedding model.

---

## RAG Generation

Retrieved knowledge is passed to Groq.

The model returns structured information such as:

```json
{
  "answer": "Grounded answer...",
  "grounded": true,
  "confidence": 0.95,
  "evidence_used": [
    "knowledge-source"
  ]
}
```

However, the model does **not** have final authority to decide whether the answer is sent automatically.

![Support RAG](screenshots/05-support-rag.png)

---

# 07 — Deterministic Grounding Gate

A deterministic safety gate decides whether an AI-generated support answer may be sent to the customer.

Auto-response requires **all** of the following:

```text
retrieval succeeded
AND
top similarity score >= 0.75
AND
AI output is well formed
AND
grounded == true
AND
confidence >= 0.70
AND
evidence_used is not empty
```

Formally:

```text
AUTO_RESPONSE ⇔
    retrieval_success
    ∧ top_score >= 0.75
    ∧ valid_ai_output
    ∧ grounded == true
    ∧ confidence >= 0.70
    ∧ evidence_used exists
```

If any requirement fails:

```text
resolution_path = human_handoff
```

This prevents the LLM from independently deciding that its own answer is safe enough to send.

---

# Support Auto-Response Example

```json
{
  "status": "answered",
  "grounded": true,
  "confidence": 0.95,
  "resolution_path": "auto_response",
  "requires_human": false,
  "support_response_sent": true,
  "support_handoff_alert_sent": false
}
```

---

# Human Handoff Example

```json
{
  "status": "human_review_required",
  "answer": "Thanks for reaching out. This question needs a closer look from our support team, and someone will follow up with you shortly.",
  "grounded": false,
  "confidence": 0,
  "resolution_path": "human_handoff",
  "requires_human": true,
  "support_response_sent": false,
  "support_handoff_alert_sent": true
}
```

The unsupported AI response is not sent to the customer.

Instead:

```text
Customer
→ Safe fallback response

Internal Support Team
→ Handoff alert with diagnostic context
```

![Human Handoff](screenshots/06-human-handoff.png)

---

# Reliability & Idempotency

The project uses multiple layers of duplicate protection.

---

## Intake Deduplication

Used to prevent repeated webhook requests from re-entering the business pipeline.

```text
email + message
→ normalized
→ hashed
→ lookup
```

---

## Lead Email Tracking

```text
request_id
+ email
+ lead_category
```

---

## HIGH Sales Alert Tracking

```text
request_id
+ email
+ high
+ sales_alert
```

---

## Support Auto-Response Tracking

```text
request_id
+ email
+ support_auto_response
```

---

## Support Handoff Tracking

```text
request_id
+ email
+ support_handoff
```

These controls are designed to reduce duplicate actions across retries and repeated requests.

Because external systems can experience ambiguous network failures, this project does **not** claim a mathematically guaranteed exactly-once delivery model.

---

# Retry Strategy

Retries are configured according to **side-effect safety**.

---

## Native Retry Enabled

Examples of operations where retry is relatively safe:

- CRM contact search
- CRM contact read
- Idempotent contact update
- Existing opportunity lookup
- Pinecone retrieval
- Gemini embedding generation
- Groq model calls
- Data Table lookup operations

---

## Native Retry Disabled

Blind retry is intentionally avoided for side-effecting operations such as:

- Creating a CRM contact
- Creating a CRM opportunity
- Sending Gmail messages
- Inserting tracking records

Why?

Consider:

```text
API processes request successfully
        ↓
Network response is lost
        ↓
Client assumes failure
        ↓
Blind automatic retry
        ↓
Duplicate side effect
```

For this reason, potentially non-idempotent writes fail safely instead of being retried blindly.

---

# Error Handling

The workflow includes defensive paths for several failure classes.

---

## Invalid Input

Rejected before AI or CRM processing.

---

## Duplicate Input

Stopped before further processing.

---

## Intent Classification Failure

Malformed or unsafe output is normalized into a safe state.

---

## Lead Qualification Failure

Malformed AI qualification output does not automatically become a valid lead score.

---

## CRM Failure

CRM errors are:

```text
Caught
→ Sanitized
→ Logged
→ Routed to Safe Review Response
```

---

## Retrieval Failure

If Pinecone retrieval fails:

```text
requires_human = true
resolution_path = human_handoff
```

---

## RAG Generation Failure

Malformed or failed AI generation cannot automatically send a customer answer.

---

## Email Failure

Tracking is recorded only after a successful API send response.

---

# Operational Logging & Observability

A centralized `automation_event_log` Data Table provides request-level observability.

Examples of logged workflow areas include:

```text
intake
intent_classification
sales_qualification
crm
customer_followup_email
sales_alert
support_retrieval
support_generation
support_delivery
```

---

## Operational Log Structure

| Field | Purpose |
|---|---|
| `event_id` | Unique event identifier |
| `request_id` | Correlates events belonging to the same request |
| `email` | Associated request email |
| `workflow_area` | Functional workflow section |
| `event_type` | Specific operation/event |
| `status` | Success / failure / state |
| `severity` | Info / warning / error |
| `node_name` | Originating node |
| `error_code` | Sanitized error classification |
| `error_message` | Sanitized diagnostic message |
| `retryable` | Indicates whether retry may be appropriate |
| `attempt` | Attempt number |
| `timestamp` | Event time |

Sensitive API credentials, tokens, prompts, or full RAG context are not intentionally written to this operational table.

---

# Data Tables

The workflow uses seven n8n Data Tables.

| Data Table | Purpose |
|---|---|
| `lead_support_intake_dev` | Request-level intake deduplication |
| `email_send_tracking` | Lead follow-up email tracking |
| `sales_alert_tracking` | HIGH sales alert tracking |
| `support_query_log` | Support retrieval and resolution audit |
| `support_response_tracking` | Support auto-response delivery tracking |
| `support_handoff_tracking` | Human-handoff alert tracking |
| `automation_event_log` | Central operational event logging |

---

# Support Query Logging

`support_query_log` stores information such as:

| Field | Example |
|---|---|
| Request ID | Unique support request |
| Query | Customer support question |
| Top retrieval score | `0.8356` |
| Grounded | `true` |
| Confidence | `0.95` |
| Resolution | `auto_response` |
| Requires human | `false` |
| Error status | Retrieval / AI error when applicable |

This creates an audit trail for why a support question was answered automatically or escalated.

---

# Technology Stack

| Technology | Purpose |
|---|---|
| **n8n** | Workflow orchestration |
| **Groq** | Intent classification, lead qualification, RAG answer generation |
| **Google Gemini** | Embedding generation |
| **Pinecone** | Vector database and semantic retrieval |
| **HighLevel / GoHighLevel** | CRM contact and opportunity management |
| **Gmail API** | Customer follow-ups and internal alerts |
| **n8n Data Tables** | Deduplication, state tracking, and logging |
| **JavaScript / Code Nodes** | Validation, scoring, parsing, error handling |
| **REST APIs** | External-service integrations |
| **JSON** | Structured workflow communication |

---

# AI Models Used

## Groq

```text
openai/gpt-oss-20b
```

Used for:

- Intent classification
- Lead qualification signal extraction
- RAG answer generation

---

## Google Gemini Embeddings

```text
models/gemini-embedding-2
```

Used for:

- Customer-support query embeddings
- Knowledge-base vector embeddings

Current vector dimension:

```text
3072
```

---

# DEV End-to-End Testing

The project has been tested end-to-end in a controlled DEV environment.

---

## Functional Test Matrix

| Scenario | Expected Behavior | Result |
|---|---|---|
| Invalid payload | Reject request | PASS |
| General inquiry | Route to General | PASS |
| HIGH sales lead | Score 8–10 and process HIGH path | PASS |
| MEDIUM sales lead | Score 5–7 and process MEDIUM path | PASS |
| LOW sales lead | Score 1–4 and process LOW path | PASS |
| Duplicate intake | Stop before AI/CRM/email | PASS |
| CRM contact resolution | Reuse/update existing contact where applicable | PASS |
| Existing opportunity | Reuse instead of duplicate creation | PASS |
| HIGH follow-up email | Send customer email | PASS |
| HIGH internal alert | Send internal sales alert | PASS |
| MEDIUM follow-up | Send MEDIUM customer email | PASS |
| LOW follow-up | Send LOW customer email | PASS |
| Known support question | Generate grounded answer | PASS |
| Support auto-response | Send approved customer answer | PASS |
| Unsupported support question | Escalate to human | PASS |
| Human handoff alert | Notify internal support | PASS |
| Operational logging | Record lifecycle events | PASS |

---

# Lead Qualification Test Results

Controlled DEV tests produced:

| Lead Type | Score | Category | Result |
|---|---:|---|---|
| Strong purchase lead | 10/10 | HIGH | PASS |
| Moderate interest lead | 6/10 | MEDIUM | PASS |
| Early-stage future lead | 2/10 | LOW | PASS |

---

# RAG Test Results

Representative DEV support tests:

| Query Type | Retrieval Score | Grounded | Confidence | Resolution |
|---|---:|---|---:|---|
| Known password-reset question | `0.8356` | Yes | `0.95` | Auto Response |
| Unsupported security-key question | `0.6440` | No | `0` | Human Handoff |

This demonstrates that the deterministic gate can distinguish between a well-supported knowledge-base answer and an unsupported question.

---

# Additional Defensive Testing

The support pipeline was also tested for:

- Retrieval failure
- Malformed Groq output
- Auto-response idempotency replay
- Auto-response email failure path
- Handoff idempotency replay
- Internal handoff email failure

These failure paths were designed to end in safe states rather than uncontrolled customer-facing actions.

---

# Repository Structure

```text
AI-Lead-Customer-Support-Automation/
│
├── README.md
│
├── workflow/
│   └── AI-Lead-Support-Automation-DEV.json
│
├── docs/
│   └── Documentation-Package.md
│
└── screenshots/
    ├── 00-workflow-overview.png
    ├── 01-intake-validation.png
    ├── 02-deduplication-intent-classification.png
    ├── 03-ai-lead-scoring-crm.png
    ├── 04-followup-sales-alerts.png
    ├── 05-support-rag.png
    └── 06-human-handoff.png
```

---

# Installation & Setup

> The public workflow export is sanitized. Credentials and environment-specific IDs are not included.

---

## Step 1 — Import the Workflow

In n8n:

```text
Workflows
→ Import from File
→ Select:

workflow/AI-Lead-Support-Automation-DEV.json
```

Keep the imported workflow **inactive** initially.

---

## Step 2 — Configure Credentials

Create or attach your own credentials for:

- Groq
- Google Gemini
- Pinecone
- HighLevel CRM
- Gmail OAuth2

Never commit credentials to GitHub.

---

## Step 3 — Configure Pinecone

Create a Pinecone index compatible with the embedding configuration.

Recommended configuration matching this implementation:

| Setting | Value |
|---|---|
| Type | Dense |
| Dimension | `3072` |
| Metric | `cosine` |
| Embedding model | `gemini-embedding-2` |

Then ingest your own business knowledge base.

---

## Step 4 — Configure HighLevel CRM

Replace environment-specific values with your own:

```text
HighLevel Location / Sub-account

Pipeline ID

Pipeline Stage ID

Opportunity Configuration
```

---

## Step 5 — Create Data Tables

Create the required n8n Data Tables:

```text
lead_support_intake_dev

email_send_tracking

sales_alert_tracking

support_query_log

support_response_tracking

support_handoff_tracking

automation_event_log
```

Map the corresponding Data Table nodes to the newly created tables.

---

## Step 6 — Configure Email Recipients

Replace placeholder addresses for:

```text
Customer email sender

Internal HIGH sales alert recipient

Internal support handoff recipient
```

---

## Step 7 — Load the Knowledge Base

Replace the development knowledge base with business-specific documentation such as:

- FAQs
- Support policies
- Product documentation
- Refund policies
- Account instructions
- Service information

Generate Gemini embeddings and insert the documents into Pinecone.

---

## Step 8 — Test with the Webhook Test URL

Before activation, test:

```text
Invalid payload

General request

HIGH lead

MEDIUM lead

LOW lead

Duplicate request

Known support question

Unsupported support question
```

---

## Step 9 — Verify Logs

Check:

```text
automation_event_log

support_query_log

email_send_tracking

sales_alert_tracking

support_response_tracking

support_handoff_tracking
```

---

## Step 10 — Perform Client UAT

Before production deployment, perform User Acceptance Testing using the client's:

- CRM configuration
- Email account
- Business rules
- Knowledge base
- Escalation recipients
- Representative sales messages
- Representative customer-support questions

---

# Production Deployment Checklist

Before using this workflow for a real client:

- [ ] Create a separate PROD workflow
- [ ] Keep DEV workflow unchanged
- [ ] Use client-owned credentials
- [ ] Use client-specific CRM location
- [ ] Configure client pipeline and stages
- [ ] Configure production Data Tables
- [ ] Replace development knowledge-base content
- [ ] Rebuild Pinecone knowledge base
- [ ] Validate email templates
- [ ] Validate internal notification recipients
- [ ] Test existing CRM contact behavior
- [ ] Test new CRM contact behavior
- [ ] Test existing opportunity behavior
- [ ] Test duplicate intake behavior
- [ ] Test customer email failure
- [ ] Test support retrieval failure
- [ ] Test unsupported support question
- [ ] Test human escalation
- [ ] Evaluate RAG threshold with representative queries
- [ ] Review privacy and compliance requirements
- [ ] Perform staging/UAT
- [ ] Monitor initial production executions

---

# Security & Privacy

This public repository is intentionally sanitized.

The repository should not contain:

- API keys
- OAuth tokens
- Passwords
- Credential IDs
- Personal test email addresses
- Customer information
- HighLevel Location IDs
- HighLevel Pipeline IDs
- HighLevel Stage IDs
- n8n Data Table IDs
- Private webhook URLs
- Internal API endpoints
- Private knowledge-base content
- Production secrets

The workflow export uses placeholders or redacted values where appropriate.

---

# Public Repository Safety

Before every public release:

```text
Search repository for:

api_key
token
secret
authorization
bearer
password
@gmail.com
webhook
locationId
pipelineId
stageId
credentialId
```

Review all matches before publishing.

---

# Design Principles

This project follows several important automation engineering principles.

### 1. AI should not blindly control critical business logic

AI extracts meaning.

Deterministic code controls important decisions.

---

### 2. Customer-facing AI should be grounded

A plausible answer is not enough.

The system checks:

```text
retrieval
+ similarity
+ grounding
+ confidence
+ evidence
```

before auto-response.

---

### 3. Uncertainty should escalate safely

When evidence is insufficient:

```text
Do not guess.

Escalate to a human.
```

---

### 4. Side effects require careful retry design

A GET request and an email send should not have the same retry policy.

---

### 5. Duplicate prevention should exist at multiple levels

The system protects:

- Intake processing
- Lead emails
- Sales alerts
- Support responses
- Support handoffs

---

### 6. Automation must be observable

Every important request should be traceable through logs.

---

### 7. Failure should be controlled

External API failures should result in:

```text
Safe fallback
+ Structured logging
+ Human visibility
```

not uncontrolled workflow crashes.

---

# Known Limitations

This project intentionally documents its current limitations.

### Native Retry Granularity

n8n native retry behavior does not provide full application-level conditional retry control based on every HTTP status and side-effect scenario.

For this reason, retries are enabled or disabled according to operation safety.

---

### Ambiguous External API Outcomes

If a third-party API completes a side effect but the network response is lost, absolute exactly-once guarantees cannot always be made.

This is especially relevant to:

- Contact creation
- Opportunity creation
- Email sending

---

### CRM Partial Failure Reconciliation

If one CRM operation succeeds and a later operation fails, full automatic reconciliation is not yet implemented.

Operational logging allows the failure to be reviewed.

---

### Intent Parser Consistency

The intent-classification stage currently uses a Structured Output Parser, while some later AI stages use plain JSON plus defensive parsing.

Both approaches are functional, but the parsing strategy could be standardized in a future version.

---

### RAG Threshold Is Environment-Specific

The current DEV retrieval threshold is:

```text
0.75
```

This should not automatically be assumed optimal for every client.

Each production knowledge base should be evaluated with representative queries before selecting the final threshold.

---

# Future Improvements

Potential enhancements include:

- Automatic CRM reconciliation
- Dead-letter/manual-retry queue
- Slack notifications
- Microsoft Teams integration
- WhatsApp support
- Voice AI integration
- Ticketing-system integration
- Human support dashboard
- SLA monitoring
- Operational analytics dashboard
- Lead conversion analytics
- Centralized configuration management
- RAG evaluation dataset
- Automated threshold evaluation
- AI answer-quality scoring
- Multi-channel intake
- Multi-tenant architecture
- Production monitoring alerts
- Automated deployment pipelines

---

# Suitable Use Cases

This architecture can be adapted for:

- SaaS businesses
- Marketing agencies
- Consulting companies
- AI automation agencies
- B2B service businesses
- Lead generation companies
- Sales operations
- Customer-support automation
- CRM automation
- Helpdesk systems
- Internal business-process automation

---

# Project Completion Status

| Component | Status |
|---|---|
| Architecture | ✅ Complete |
| Intake Validation | ✅ Complete |
| Duplicate Protection | ✅ Complete |
| Intent Classification | ✅ Complete |
| AI Lead Qualification | ✅ Complete |
| Deterministic Lead Scoring | ✅ Complete |
| HighLevel CRM Integration | ✅ Complete |
| CRM Opportunity Handling | ✅ Complete |
| Lead Follow-Up | ✅ Complete |
| HIGH Sales Alert | ✅ Complete |
| Gemini Embeddings | ✅ Complete |
| Pinecone Retrieval | ✅ Complete |
| RAG Generation | ✅ Complete |
| Deterministic Grounding Gate | ✅ Complete |
| Customer Auto-Response | ✅ Complete |
| Human Handoff | ✅ Complete |
| Operational Logging | ✅ Complete |
| Failure Handling | ✅ Complete |
| DEV Functional E2E Testing | ✅ Passed |
| Public Export Sanitization | ✅ Complete |
| Portfolio Screenshots | ✅ Complete |
| Client-Specific Production Deployment | ⚙ Configuration Required |

---

# Production Readiness Statement

This workflow demonstrates **production-oriented reliability and safety patterns**.

It should not be deployed unchanged into a client production environment.

A production deployment requires:

```text
Client credentials
+ Client CRM configuration
+ Client knowledge base
+ Client business rules
+ Environment-specific Data Tables
+ Security review
+ Representative RAG evaluation
+ UAT
+ Controlled deployment
+ Monitoring
```

---

# Author

## Aminul Islam

**AI Automation Specialist | Oracle APEX & Database Developer**

Areas of focus:

- AI Workflow Automation
- n8n Automation
- AI Agents
- RAG Systems
- CRM Automation
- API Integration
- Business Process Automation
- Oracle APEX
- Oracle Database
- SQL
- PL/SQL

### Portfolio

https://www.aminulaicloud.com/

---

# About This Repository

This repository is a **sanitized portfolio and reference implementation**.

It demonstrates architecture, engineering decisions, defensive automation patterns, AI orchestration, CRM integration, and RAG safety design.

It does not contain client production credentials, customer data, private knowledge-base documents, or live environment secrets.

---

# Disclaimer

This project is provided as a technical reference and portfolio demonstration.

External API behavior, AI model behavior, CRM configuration, email delivery, vector retrieval quality, and business requirements vary between environments.

The workflow should therefore be configured, tested, security-reviewed, and validated for the target business environment before production deployment.

---

# Contact

If you are interested in:

- AI automation
- n8n workflow development
- CRM automation
- RAG-based customer support
- AI agents
- API integration
- Business process automation

you can learn more about my work at:

**https://www.aminulaicloud.com/**

---

## ⭐ Project Summary

```text
One Webhook
      ↓
Intelligent Triage
      ↓
┌────────────────────┬───────────────────────┐
│                    │                       │
│       SALES        │       SUPPORT         │
│                    │                       │
│ AI Qualification   │ Semantic Retrieval    │
│ Lead Scoring       │ RAG Generation        │
│ CRM Automation     │ Grounding Gate        │
│ Follow-Up          │ Auto Reply / Handoff  │
│ Sales Alert        │                       │
│                    │                       │
└──────────┬─────────┴──────────┬────────────┘
           │                    │
           └─────────┬──────────┘
                     ↓
          Reliability + Observability
```

**Built to demonstrate how AI can be combined with deterministic engineering controls for safer, more reliable business automation.**