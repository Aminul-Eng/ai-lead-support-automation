# AI-Powered Lead & Customer Support Automation
## Technical Documentation Package

> Technical architecture, business logic, reliability controls, AI safety mechanisms, integration design, testing evidence, and production deployment considerations for the n8n-based AI Lead & Customer Support Automation System.

---

## Document Status

| Item | Status |
|---|---|
| Environment | DEV |
| Workflow State | Inactive / Unpublished |
| Core Development | Complete |
| Functional DEV E2E Testing | Passed |
| Public Export | Sanitized |
| Portfolio Documentation | Complete |
| Client Production Deployment | Requires client-specific configuration and UAT |

This document describes the current tested DEV implementation.

The workflow demonstrates **production-oriented engineering patterns**, but it should not be deployed unchanged into a client production environment.

---

# 1. Project Overview

The **AI-Powered Lead & Customer Support Automation System** is an n8n workflow designed to process both sales inquiries and customer-support requests through a single automated intake channel.

The system performs:

- Input normalization and validation
- Request deduplication
- AI intent classification
- AI-assisted lead qualification
- Deterministic lead scoring
- HighLevel CRM synchronization
- Opportunity deduplication
- Customer follow-up emails
- HIGH-priority internal sales alerts
- Retrieval-Augmented Generation (RAG)
- Deterministic grounding validation
- Customer support auto-response
- Human handoff
- Idempotency protection
- Operational event logging
- Defensive error handling

The architecture combines **LLM reasoning with deterministic business controls** so that AI is used for semantic understanding while critical actions remain governed by explicit rules.

---

# 2. Business Problem

Organizations receiving both sales and support requests through the same intake channel often experience:

- Manual triage delays
- Inconsistent lead qualification
- Slow sales follow-up
- Missed high-value opportunities
- Duplicate CRM records
- Duplicate customer emails
- Repetitive support workload
- AI hallucination risk
- Poor failure visibility
- Difficult troubleshooting

A basic AI chatbot alone does not solve these problems because business-critical actions require:

- State management
- Duplicate protection
- CRM integration
- Reliable routing
- Safety controls
- Human escalation
- Logging
- Failure handling

This project addresses those requirements through a modular automation architecture.

---

# 3. Solution Architecture

```text
Website / Form / Webhook
          │
          ▼
Normalize & Validate
          │
          ▼
Duplicate Detection
          │
          ▼
AI Intent Classification
          │
     ┌────┼───────────────┐
     │    │               │
     │    │               │
   SALES SUPPORT       GENERAL / UNKNOWN
     │    │               │
     │    │               └── Safe Response / Review
     │    │
     │    └── Gemini Embeddings
     │            │
     │            ▼
     │      Pinecone Retrieval
     │            │
     │            ▼
     │       Groq RAG
     │            │
     │            ▼
     │     Deterministic Gate
     │          /     \
     │         /       \
     │ Auto Response   Human Handoff
     │
     └── Groq Lead Qualification
              │
              ▼
       Deterministic Scoring
              │
              ▼
       HIGH / MEDIUM / LOW
              │
              ▼
         HighLevel CRM
              │
              ▼
       Customer Follow-Up
              │
              ▼
        HIGH Sales Alert

All major stages
        │
        ▼
Operational Event Logging
```

---

# 4. Workflow Modules

The workflow is organized into eight logical modules.

| Module | Name | Responsibility |
|---|---|---|
| 01 | Intake & Validation | Receives, normalizes, and validates inbound requests |
| 02 | Deduplication & Intent Classification | Prevents duplicate processing and identifies request intent |
| 03 | AI-Assisted Lead Scoring | Extracts semantic sales signals and calculates deterministic lead scores |
| 04 | CRM Integration | Resolves HighLevel contacts and HIGH-lead opportunities |
| 05 | Follow-Up & Sales Alerts | Sends tiered customer emails and HIGH-priority internal alerts |
| 06 | Retrieval-Augmented Support Generation | Retrieves knowledge and generates grounded support answers |
| 07 | Deterministic Grounding Gate & Human Handoff | Controls whether AI may answer automatically |
| 08 | Operational Logging & Error Handling | Provides centralized event visibility and safe failure handling |

Module 08 is a **logical cross-cutting layer**. Its logging nodes remain distributed near the business operations they observe rather than being physically centralized on the canvas.

---

# 5. Intake & Validation

The workflow accepts inbound requests through an n8n webhook.

Typical normalized fields include:

```json
{
  "request_id": "generated-request-id",
  "source": "website",
  "name": "Customer Name",
  "email": "customer@example.com",
  "phone": "",
  "company": "Example Company",
  "job_title": "Operations Manager",
  "budget": "$5000",
  "message": "We need help automating our lead process.",
  "received_at": "timestamp"
}
```

Required validation includes:

- Name presence
- Valid email format
- Message presence

Invalid payloads are rejected before:

- AI processing
- CRM calls
- Vector retrieval
- Email delivery

Example validation result:

```json
{
  "valid": false,
  "validation_errors": [
    "name is required",
    "email format is invalid",
    "message is required"
  ]
}
```

---

# 6. Request Deduplication

The intake layer protects the workflow from repeated processing.

A deterministic message hash / idempotency key is generated from normalized:

```text
email + message
```

The key is checked against:

```text
lead_support_intake_dev
```

If a matching request already exists, processing stops before expensive or side-effecting operations.

Example:

```json
{
  "status": "duplicate",
  "message": "This request has already been received.",
  "duplicate": true
}
```

This prevents the same inbound request from unnecessarily reaching:

- LLM classification
- CRM
- Email
- RAG processing

---

# 7. Intent Classification

Groq is used to classify each accepted request.

## Model

```text
openai/gpt-oss-20b
```

The intent stage currently uses an n8n **Structured Output Parser**.

Supported classifications:

| Intent | Meaning |
|---|---|
| `sales` | Potential business / automation opportunity |
| `support` | Customer support question |
| `general` | General information request |
| `unknown` | Uncertain or low-confidence request |

The classification output includes:

- Intent
- Confidence
- Summary
- Human-review indicator

Low-confidence or malformed classification output falls back to a safe state rather than forcing a downstream business decision.

---

# 8. Lead Qualification Architecture

Requests classified as `sales` enter the lead qualification pipeline.

The system deliberately separates:

```text
Semantic interpretation
        +
Deterministic scoring
```

The LLM extracts business signals.

Code calculates the final score.

This prevents the model from arbitrarily assigning a lead score.

---

# 9. AI Lead Signals

Groq evaluates five boolean signals.

| Signal | Weight |
|---|---:|
| `clear_business_need` | +2 |
| `specific_automation_need` | +2 |
| `urgency_or_timeline` | +1 |
| `pricing_demo_or_call_request` | +1 |
| `strong_purchase_intent` | +1 |

Maximum AI-derived contribution:

```text
7 points
```

The AI output is plain JSON and passes through defensive parsing.

The parser:

- Removes markdown fences when present
- Extracts the JSON object
- Uses `JSON.parse`
- Normalizes boolean values
- Rejects malformed structures
- Falls back to review on invalid output

No `eval()`-style execution is used.

---

# 10. Deterministic Lead Signals

Three additional signals are calculated independently in code.

| Signal | Weight |
|---|---:|
| Decision-maker job title | +1 |
| Budget provided | +1 |
| High budget | +1 |

The DEV high-budget threshold is:

```text
>= $10,000
```

Maximum total score:

```text
10
```

---

# 11. Lead Categories

| Score | Category |
|---:|---|
| 8–10 | HIGH |
| 5–7 | MEDIUM |
| 1–4 | LOW |

Malformed qualification output does not become a valid score.

Instead:

```text
requires_human = true
```

and the request follows a safe review path.

---

# 12. HighLevel CRM Integration

HighLevel / GoHighLevel is used for:

- Contact lookup
- Contact creation
- Contact update
- Opportunity lookup
- Opportunity creation

The CRM pipeline follows:

```text
Search Contact by Email
        │
        ▼
Resolve Exact Match
        │
   ┌────┴────┐
   │         │
Existing    Missing
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

---

# 13. HIGH Lead Opportunity Logic

HIGH leads receive additional CRM processing.

```text
HIGH Lead
   │
   ▼
Check Existing Opportunity
   │
   ├── Exists → Reuse
   │
   └── Missing → Create
```

This protects against unnecessary opportunity duplication during normal processing.

MEDIUM and LOW leads do not automatically create HIGH-priority opportunities.

---

# 14. Lead Follow-Up Automation

Each qualified lead category has a dedicated customer follow-up path.

## HIGH

```text
CRM
→ HIGH Customer Email
→ Internal Sales Alert
→ Final Response
```

## MEDIUM

```text
CRM
→ MEDIUM Customer Email
→ Final Response
```

## LOW

```text
CRM
→ LOW Customer Email
→ Final Response
```

---

# 15. Lead Email Idempotency

Customer follow-up emails use a check-before-send strategy.

The logical tracking key is based on:

```text
request_id
+ email
+ lead_category
```

Flow:

```text
Compute Event Key
      │
      ▼
Check Tracking Table
      │
  ┌───┴────┐
  │        │
Exists    Missing
  │        │
Skip      Send Gmail
           │
           ▼
      Send Successful?
        /       \
      Yes       No
       │         │
       ▼         ▼
Record Sent   Mark Failed
```

A tracking row is inserted only after Gmail reports a successful send.

This is designed to reduce duplicate sends.

It is not an absolute exactly-once delivery guarantee across third-party systems.

---

# 16. HIGH Sales Alert

HIGH leads also trigger an internal sales notification.

The alert has its own tracking key:

```text
request_id
+ email
+ high
+ sales_alert
```

This separates customer-email idempotency from internal-alert idempotency.

Customer email failure does not inherently prevent the internal HIGH sales alert path from being evaluated.

---

# 17. Support RAG Architecture

Requests classified as `support` enter the RAG pipeline.

```text
Normalize Support Query
        │
        ▼
Generate Gemini Embedding
        │
        ▼
Pinecone Vector Retrieval
        │
        ▼
Validate Retrieved Context
        │
        ▼
Has Usable Context?
     /         \
   Yes          No
    │            │
    ▼            ▼
Groq RAG      Skip Generation
Generation       │
    │            │
    ▼            │
Validate Answer  │
    │            │
    └──────┬─────┘
           ▼
Determine Support Resolution
```

---

# 18. Embedding Configuration

## Provider

Google Gemini

## Model

```text
models/gemini-embedding-2
```

## Vector Dimension

```text
3072
```

## Pinecone Configuration

| Setting | DEV Value |
|---|---|
| Index type | Dense |
| Dimension | `3072` |
| Similarity metric | Cosine |
| Retrieval count | Top 4 |

The DEV implementation uses an environment-specific Pinecone index.

Production deployments should use a client/environment-specific index rather than reusing the development index.

---

# 19. RAG Generation

Groq generates the support answer from retrieved context.

Model:

```text
openai/gpt-oss-20b
```

The RAG stage uses **plain JSON + defensive parsing**, not the Structured Output Parser used by the intent stage.

Expected semantic structure:

```json
{
  "answer": "Support response",
  "grounded": true,
  "confidence": 0.95,
  "evidence_used": [
    "knowledge-source"
  ],
  "requires_human": false
}
```

Malformed AI output falls back to a safe ungrounded state.

---

# 20. Deterministic Grounding Gate

The LLM is not allowed to independently decide whether its response is safe to send.

Auto-response requires all conditions below:

```text
retrieval succeeded
AND
top_score >= 0.75
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
    retrieval_ok
    ∧ top_score >= 0.75
    ∧ valid_ai_output
    ∧ grounded == true
    ∧ confidence >= 0.70
    ∧ evidence_used != empty
```

If any condition fails:

```text
resolution_path = human_handoff
requires_human = true
```

This deterministic gate is one of the main AI-safety controls in the architecture.

---

# 21. Support Auto-Response

If the grounding gate passes:

```text
Compute Support Response Key
        │
        ▼
Check Already Sent
        │
        ▼
Send Customer Answer
        │
        ▼
Confirm Send
        │
        ▼
Record Tracking Row
```

Tracking key:

```text
request_id
+ email
+ support_auto_response
```

---

# 22. Human Handoff

If the deterministic gate fails:

```text
Compute Handoff Key
        │
        ▼
Check Existing Handoff
        │
        ▼
Send Internal Support Alert
        │
        ▼
Confirm Alert
        │
        ▼
Record Tracking Row
```

Tracking key:

```text
request_id
+ email
+ support_handoff
```

The customer receives a generic safe response such as:

```text
Thanks for reaching out. This question needs a closer look
from our support team, and someone will follow up with you shortly.
```

The customer is not sent an unsupported AI-generated answer.

Internal support receives the diagnostic handoff context.

---

# 23. Data Tables

Seven n8n Data Tables provide persistence, idempotency, auditability, and operational state.

| Data Table | Responsibility |
|---|---|
| `lead_support_intake_dev` | Request deduplication |
| `email_send_tracking` | HIGH/MEDIUM/LOW customer email tracking |
| `sales_alert_tracking` | HIGH internal sales-alert tracking |
| `support_query_log` | RAG retrieval and resolution audit |
| `support_response_tracking` | Support auto-response tracking |
| `support_handoff_tracking` | Human-handoff alert tracking |
| `automation_event_log` | Central operational event log |

Environment-specific Data Table IDs are not included in the public documentation.

---

# 24. Support Query Audit

`support_query_log` captures information used to understand support decisions.

Typical fields include:

```text
request_id
email
query
top_score
grounded
confidence
resolution_path
requires_human
timestamp
error_status
```

This allows an operator to determine why a request:

```text
Auto-Responded
```

or

```text
Escalated to Human Support
```

---

# 25. Operational Logging

The `automation_event_log` table provides centralized observability.

Schema:

| Field | Description |
|---|---|
| `event_id` | Unique event identifier |
| `request_id` | Request correlation identifier |
| `email` | Associated request email |
| `workflow_area` | Functional workflow area |
| `event_type` | Specific operation |
| `status` | Result/state |
| `severity` | Info, warning, or error |
| `node_name` | Originating workflow node |
| `error_code` | Sanitized error classification |
| `error_message` | Sanitized diagnostic information |
| `retryable` | Whether retry may be safe |
| `attempt` | Execution attempt |
| `timestamp` | Event timestamp |

Examples of `workflow_area`:

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

# 26. Retry Strategy

n8n native retry is not sufficiently status-aware to distinguish every transient and permanent failure scenario automatically.

For this reason, retry behavior is based primarily on **side-effect safety**.

---

## Retry Enabled

Examples:

- CRM search
- CRM read
- Idempotent contact update
- Existing opportunity lookup
- Pinecone retrieval
- Gemini embedding generation
- Groq LLM calls
- Data Table lookups

Typical configuration:

```text
retryOnFail = true
maxTries = 3
waitBetweenTries = 1000 ms
```

---

## Blind Retry Disabled

Examples:

- Create Contact
- Create Opportunity
- Gmail sends
- Data Table inserts

Reason:

```text
Third-party operation succeeds
        │
        ▼
Network response is lost
        │
        ▼
Client sees failure
        │
        ▼
Blind retry
        │
        ▼
Potential duplicate side effect
```

Failing safely is preferable to automatically duplicating a customer-facing or CRM action.

---

# 27. CRM Failure Handling

HighLevel failures are routed into a shared defensive path.

```text
HighLevel Error
      │
      ▼
Handle CRM Failure
      │
      ▼
Classify Error
      │
      ▼
Sanitize Error
      │
      ▼
Log CRM Failure
      │
      ▼
Safe Review Response
```

The public response does not expose:

- Raw provider error
- Stack trace
- API credentials
- Tokens
- Internal URLs
- Sensitive node configuration

---

# 28. Error Sanitization

Operational error handling removes or redacts sensitive patterns such as:

```text
key
token
secret
authorization
bearer
URLs
```

`automation_event_log` intentionally avoids storing:

- API credentials
- Full prompts
- Full RAG context
- Raw provider stack traces

---

# 29. AI Failure Handling

Each AI stage has a safe fallback strategy.

## Intent Classification

Failure:

```text
unknown
→ requires review
```

## Lead Qualification

Failure:

```text
review
→ requires_human = true
```

## RAG Generation

Failure:

```text
grounded = false
confidence = 0
→ human_handoff
```

The workflow therefore prefers safe degradation over forced AI output.

---

# 30. Security & Privacy Controls

The public portfolio export is sanitized.

It intentionally excludes:

- API keys
- OAuth tokens
- Passwords
- Credential IDs
- Personal test data
- Customer information
- HighLevel Location IDs
- HighLevel Pipeline IDs
- HighLevel Stage IDs
- n8n Data Table IDs
- Private webhook URLs
- Private knowledge-base documents

Authentication is expected to be configured through the target n8n environment's credential store.

---

# 31. DEV Functional End-to-End Testing

The workflow has been tested end-to-end in a controlled DEV environment using the connected integrations.

The tests exercised actual workflow paths involving:

- Groq
- Google Gemini
- Pinecone
- HighLevel
- Gmail
- n8n Data Tables

---

# 32. Functional Test Matrix

| Test Scenario | Expected Result | Status |
|---|---|---|
| Invalid input | Rejected before business processing | PASS |
| General intent | Routed to general response | PASS |
| HIGH sales lead | HIGH qualification path | PASS |
| MEDIUM sales lead | MEDIUM qualification path | PASS |
| LOW sales lead | LOW qualification path | PASS |
| Duplicate request | Stop before AI/CRM/email | PASS |
| CRM contact resolution | Existing contact resolved | PASS |
| Existing opportunity | Reused instead of duplicated | PASS |
| HIGH customer email | Sent | PASS |
| HIGH internal sales alert | Sent | PASS |
| MEDIUM customer email | Sent | PASS |
| LOW customer email | Sent | PASS |
| Known support question | Grounded response | PASS |
| Support auto-response | Customer response sent | PASS |
| Unsupported support question | Human handoff | PASS |
| Internal support alert | Sent | PASS |
| Operational logging | Events recorded | PASS |

---

# 33. Lead Qualification Test Evidence

Controlled DEV tests produced:

| Test | Score | Category |
|---|---:|---|
| Strong qualified lead | 10/10 | HIGH |
| Moderate-interest lead | 6/10 | MEDIUM |
| Early-stage future lead | 2/10 | LOW |

The results demonstrate the intended category boundaries:

```text
HIGH   = 8–10
MEDIUM = 5–7
LOW    = 1–4
```

---

# 34. Duplicate Protection Test

A previously processed HIGH lead payload was submitted again.

Result:

```json
{
  "status": "duplicate",
  "message": "This request has already been received.",
  "duplicate": true
}
```

The duplicate was detected before the request re-entered:

```text
AI
CRM
Email
```

---

# 35. RAG Auto-Response Test

Known support query:

```text
How do I reset my password?
```

Observed result:

| Metric | Result |
|---|---|
| Top retrieval score | `0.835647583` |
| Grounded | `true` |
| Confidence | `0.95` |
| Resolution | `auto_response` |
| Customer response sent | `true` |
| Human alert sent | `false` |

This satisfied the deterministic grounding gate.

---

# 36. Human Handoff Test

Unsupported security-related query produced:

| Metric | Result |
|---|---|
| Top retrieval score | `0.643966734` |
| Grounded | `false` |
| Confidence | `0` |
| Resolution | `human_handoff` |
| Customer AI answer sent | `false` |
| Internal handoff alert | `true` |

The workflow therefore prevented an unsupported AI answer from being sent automatically.

---

# 37. Defensive Failure Testing

The support architecture has also been exercised against controlled failure scenarios including:

- Pinecone retrieval failure
- Malformed Groq output
- Auto-response idempotency replay
- Auto-response Gmail failure
- Human-handoff idempotency replay
- Handoff-alert Gmail failure

Expected safe-state behavior was observed in these controlled tests.

---

# 38. Operational Logging Verification

Operational logging was verified during the E2E test cycle.

Examples of recorded events included:

```text
validation_failed
intent_classified
lead_qualified
contact_resolved
high_lead_email
medium_lead_email
low_lead_email
high_lead_internal_alert
intake_recorded
```

Events were correlated using `request_id`.

This provides lifecycle traceability across multiple workflow modules.

---

# 39. Production Deployment Model

The current DEV workflow should remain the development/reference implementation.

For a client deployment:

```text
DEV Template
      │
      ▼
Create Client-Specific PROD Workflow
      │
      ▼
Configure Client Credentials
      │
      ▼
Configure Client CRM
      │
      ▼
Create PROD Data Tables
      │
      ▼
Load Client Knowledge Base
      │
      ▼
Evaluate RAG Threshold
      │
      ▼
Client UAT
      │
      ▼
Controlled Activation
      │
      ▼
Monitoring
```

---

# 40. Client Production Checklist

Before production activation:

- [ ] Create separate PROD workflow
- [ ] Preserve DEV workflow
- [ ] Configure client-owned credentials
- [ ] Configure client HighLevel location
- [ ] Configure client CRM pipeline
- [ ] Configure pipeline stages
- [ ] Configure customer email sender
- [ ] Configure internal sales recipient
- [ ] Configure internal support recipient
- [ ] Create production Data Tables
- [ ] Load client knowledge base
- [ ] Create production Pinecone index if required
- [ ] Validate embedding configuration
- [ ] Test invalid requests
- [ ] Test duplicates
- [ ] Test HIGH leads
- [ ] Test MEDIUM leads
- [ ] Test LOW leads
- [ ] Test existing contacts
- [ ] Test new contacts
- [ ] Test existing opportunities
- [ ] Test support known questions
- [ ] Test unsupported questions
- [ ] Test human handoff
- [ ] Test email failures
- [ ] Test CRM failures
- [ ] Review logs
- [ ] Evaluate RAG threshold using representative queries
- [ ] Review security/privacy requirements
- [ ] Complete UAT
- [ ] Activate production webhook
- [ ] Monitor initial production executions

---

# 41. Production Considerations

## Environment Separation

DEV and PROD should use separate:

- Workflows
- Data Tables
- Credentials where appropriate
- Pinecone data/index configuration
- Notification recipients

---

## Client Knowledge Base

The DEV knowledge base is demonstration content.

Production RAG must use the client's authoritative documentation.

Examples:

- FAQs
- Product documentation
- Refund policy
- Subscription policy
- Account instructions
- Support procedures

---

## RAG Threshold Evaluation

Current DEV threshold:

```text
top_score >= 0.75
```

This threshold performed correctly in the controlled test cases but should not be treated as universally optimal.

Before production deployment, evaluate representative:

```text
Known questions
Unknown questions
Borderline questions
Paraphrased questions
Ambiguous questions
```

and tune for the client's knowledge base.

The priority should be reducing unsafe false auto-responses.

---

# 42. Known Limitations

## 42.1 Native Retry Granularity

The retry policy is based on operation safety rather than a complete HTTP-status-aware retry engine.

---

## 42.2 Ambiguous Third-Party Outcomes

An external API may successfully perform an action while its response is lost due to a network failure.

Therefore the architecture does not claim an absolute exactly-once guarantee.

---

## 42.3 CRM Partial Failure Reconciliation

Example:

```text
Contact created successfully
        │
        ▼
Opportunity lookup fails
```

The failure is logged and safely handled, but automatic reconciliation of all partial CRM states is not yet implemented.

---

## 42.4 AI Parsing Strategy

Intent classification currently uses:

```text
Structured Output Parser
```

while lead qualification and RAG generation use:

```text
Plain JSON + Defensive Code Parsing
```

This is functional but could be standardized in a future release.

---

## 42.5 RAG Threshold Portability

The current `0.75` retrieval threshold is environment-specific and requires evaluation against each client's knowledge base.

---

# 43. Future Improvements

Potential improvements include:

### Reliability

- CRM reconciliation workflow
- Dead-letter queue
- Manual retry queue
- Advanced status-aware retry policy
- Failure notification automation

### Observability

- Operational dashboard
- Error-rate metrics
- Lead conversion metrics
- RAG auto-response rate
- Handoff-rate analytics
- SLA monitoring

### AI / RAG

- Formal RAG evaluation dataset
- Automated threshold benchmarking
- Response-quality scoring
- Citation/evidence presentation
- Reranking
- Query rewriting
- Knowledge-base versioning

### Channels

- WhatsApp
- Telegram
- Slack
- Microsoft Teams
- Web chat
- Voice AI
- Ticketing platforms

### Architecture

- Multi-tenant client configuration
- Centralized environment configuration
- CI/CD workflow deployment
- Automated configuration validation

---

# 44. Technology Stack

| Layer | Technology |
|---|---|
| Workflow Orchestration | n8n |
| Intent Classification | Groq |
| Lead Qualification | Groq |
| Support Generation | Groq |
| Embeddings | Google Gemini |
| Vector Database | Pinecone |
| CRM | HighLevel / GoHighLevel |
| Email | Gmail API |
| State / Tracking | n8n Data Tables |
| Business Logic | JavaScript / n8n Code Nodes |
| Integration | REST APIs / OAuth2 |
| Data Exchange | JSON |

---

# 45. Repository Structure

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

# 46. Public Repository Security Checklist

Before publishing or updating the repository, search all files for:

```text
api_key
apikey
token
secret
password
authorization
bearer
credential
@gmail.com
webhook
locationId
pipelineId
stageId
```

Review every match manually.

Do not publish:

```text
.env
credentials
OAuth tokens
private KB files
client exports
customer data
production logs
```

---

# 47. Engineering Principles

The project follows these principles:

1. **Use AI for semantic understanding, not uncontrolled business decisions.**
2. **Combine probabilistic AI with deterministic rules.**
3. **Do not automatically send unsupported RAG answers.**
4. **Escalate uncertainty to humans.**
5. **Treat side-effecting operations differently from read operations.**
6. **Protect outbound actions against duplicate execution.**
7. **Maintain request-level operational traceability.**
8. **Sanitize errors before storing or returning them.**
9. **Separate development configuration from production configuration.**
10. **Test client-specific behavior before activation.**

---

# 48. Final Technical Status

| Capability | Status |
|---|---|
| Webhook Intake | Complete |
| Input Validation | Complete |
| Request Deduplication | Complete |
| Intent Classification | Complete |
| AI Lead Qualification | Complete |
| Deterministic Lead Scoring | Complete |
| HighLevel CRM | Complete |
| Opportunity Deduplication | Complete |
| Lead Follow-Up | Complete |
| HIGH Sales Alert | Complete |
| Gemini Embeddings | Complete |
| Pinecone Retrieval | Complete |
| Groq RAG | Complete |
| Grounding Gate | Complete |
| Support Auto-Response | Complete |
| Human Handoff | Complete |
| Delivery Tracking | Complete |
| Operational Logging | Complete |
| Defensive Failure Handling | Complete |
| Controlled DEV E2E Testing | Passed |
| Public Export Sanitization | Complete |
| Client Production Deployment | Client Configuration + UAT Required |

---

# 49. Conclusion

The **AI-Powered Lead & Customer Support Automation System** demonstrates an end-to-end approach to business automation that combines:

```text
AI
+
Deterministic Business Logic
+
CRM
+
RAG
+
Human Escalation
+
Idempotency Protection
+
Operational Observability
```

The project is designed to demonstrate more than a simple AI workflow.

It shows how AI-driven automation can be engineered with explicit safety gates, duplicate protection, failure handling, auditability, and human escalation for real business use.

The current DEV implementation has successfully completed controlled functional end-to-end testing.

For production use, the workflow should be deployed into a client-specific environment with appropriate credentials, CRM configuration, business knowledge, representative RAG evaluation, security review, UAT, and monitored rollout.

---

## Author

**Aminul Islam**

AI Automation Specialist | Oracle APEX & Database Developer

Focus Areas:

- n8n Workflow Automation
- AI Agents
- RAG Systems
- CRM Automation
- API Integration
- Business Process Automation
- Oracle APEX
- Oracle Database
- SQL / PL/SQL

Portfolio:

`https://www.aminulaicloud.com/`