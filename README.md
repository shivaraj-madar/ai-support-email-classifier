# AI Customer Support Email Classifier

Automated ticket routing for customer support inboxes, built with **n8n**, **Google Gemini 2.5 Flash**, and **Gmail SMTP**.

Incoming support emails are sent to a webhook, classified by Gemini into **Billing**, **Technical**, or **Complaint**, and instantly routed to the right team by email — complete with SLA deadlines and ticket metadata. No manual triage required.

![Workflow diagram] <img width="1001" height="535" alt="image" src="https://github.com/user-attachments/assets/43b4b0df-1fcd-4133-82f5-2e98e1e2430e" />


---

## Why this exists

Support teams sorting tickets by hand run into the same problems over and over:

| Problem | Impact |
|---|---|
| ⏱️ Slow response time | Emails routed manually — hours lost sorting tickets every day |
| ❌ Misrouted tickets | Wrong team assignments → delays, duplicate work, frustrated customers |
| 📧 High email volume | Hundreds of emails daily with no automated filtering |
| 🔥 Complaint escalation | Critical complaints buried in the inbox — managers alerted too late |

This workflow removes the manual step entirely: an email comes in, Gemini reads it, and the right team gets notified within seconds.

## What it does

1. **Auto-classification** — Gemini reads each email and categorizes it as `Billing`, `Technical`, or `Complaint` in under 2 seconds.
2. **Smart routing** — IF nodes route each category to the correct team: Billing, Technical, or Manager escalation.
3. **Instant notification** — A formatted email with full ticket details (ticket ID, category, timestamp, SLA deadline) is sent automatically to the right team.

| Category | Routed to | SLA |
|---|---|---|
| Billing | Billing team | 24 hours |
| Technical | Technical team (ticket created + notified) | 48 hours |
| Complaint | Manager (escalation) | 2 hours |

## How it works

The workflow has 13 nodes in n8n: 

```
Webhook (POST /support-ticket)
        │
        ▼
  Set – Sample Input  (manual-test stand-in for real payload)
        │
        ▼
  AI – Classify Email (Gemini)   ← HTTP Request to Gemini 2.5 Flash
        │
        ▼
  Set – Extract Category         (parses category out of the AI response)
        │
        ▼
  IF – Is Billing? ──Yes──► Email – Notify Billing Team ─ ─┐
        │ No                                               │
        ▼                                                  │
  IF – Is Technical? ──Yes──► Set – Create Technical Ticket│
        │ No                          │                    │
        │                             ▼                    │
        │                   Email – Create Technical Ticket│
        │                             │                    │
        ▼                             │                    │
  Email – Escalate to Manager         │                    │
        │                             │                    │
        └─────────────┬───────────────┴────────────────────┘
                      ▼
            Webhook Response – Success

  
```

**Classification prompt** sent to Gemini:

> Classify this customer support email into exactly one of these categories: Billing, Technical, Complaint. Respond with ONLY the category name, nothing else.

`generationConfig` uses `temperature: 0` (deterministic output) and `maxOutputTokens: 200`.

A separate **Error Trigger → Email Alert** branch emails an admin if any node in the workflow fails, with the failed node name, execution ID, and error message included.

## Example runs

<table>
<tr>
<td width="33%">

**Billing**

![Billing test case] <img width="1200" height="675" alt="image" src="https://github.com/user-attachments/assets/69afd820-afa2-48b7-977a-7318fa2e8c1a" />


</td>
<td width="33%">

**Technical**

![Technical test case] <img width="1200" height="675" alt="image" src="https://github.com/user-attachments/assets/d8d7254f-dae0-4896-b374-3926e041364d" />


</td>
<td width="33%">

**Complaint**

![Complaint test case] <img width="1200" height="675" alt="image" src="https://github.com/user-attachments/assets/0ead7e74-a1bb-4604-af5d-8b549c3ba5bd" />


</td>
</tr>
</table>

## Setup

### Prerequisites

- An [n8n](https://n8n.io/) instance (self-hosted http://localhost:5678/workflow/FJeBG5XeUg6hhxkW)
- A [Google AI Studio](https://aistudio.google.com/app/apikey) API key (Gemini)
- A Gmail account with [2-Step Verification](https://myaccount.google.com/security) enabled and an [App Password](https://myaccount.google.com/apppasswords) generated for SMTP

### 1. Import the workflow

In n8n: **Workflows → Add Workflow → Import from File**, and select [`workflow/AI_Customer_Support_Email_Classifier.json`](workflow/AI_Customer_Support_Email_Classifier.json).

### 2. Configure the Gemini API key

The workflow reads the key from an environment variable rather than storing it in the workflow itself. Set it on your n8n instance:

```bash
export GEMINI_API_KEY="your-key-here"
```

Or, if you're on n8n Cloud / can't set host environment variables, open the **AI – Classify Email (Gemini)** node and replace `{{ $env.GEMINI_API_KEY }}` in the URL with your key directly (not recommended for shared instances).

> **Never commit a real API key to this repo.** See [Security](#security) below.

### 3. Configure SMTP credentials

Create an **SMTP** credential in n8n (**Credentials → New → SMTP**) with:

| Field | Value |
|---|---|
| Host | `smtp.gmail.com` |
| Port | `465` |
| SSL | enabled |
| User | your Gmail address |
| Password | your 16-character **App Password** (not your normal Gmail password) |

Then assign this credential to the four email nodes: `Email - Notify Billing Team`, `Email - Escalate to Manager`, `Email create technical ticket`, and `Email - Send Error Alert`.

### 4. Set your team email addresses

Each email node currently points to a placeholder address (`your-team-inbox@example.com`). Update the `fromEmail` / `toEmail` fields per node to your real team inboxes.

### 5. Activate and test

Activate the workflow, then send a test request to your webhook URL:

```bash
curl -X POST https://<your-n8n-instance>/webhook/support-ticket \
  -H "Content-Type: application/json" \
  -d '{"email": "My order is delayed and I want a refund"}'
```

Expected response:

```json
{
  "status": "success",
  "ticket_id": "TKT-1779116987154",
  "category": "Complaint",
  "message": "Ticket classified and routed successfully"
}
```

## Project structure

```
.
├── README.md
├── LICENSE
├── .gitignore
├── workflow/
│   └── AI_Customer_Support_Email_Classifier.json   # n8n workflow export (sanitized)
├── assets/
│   ├── workflow-diagram.jpg
│   ├── test-case-billing.jpg
│   ├── test-case-technical.jpg
│   └── test-case-complaint.jpg
└── docs/
    └── CHALLENGES.md                                # issues hit during build + fixes
```

## Security

This repository's workflow JSON has been sanitized:

- The Gemini API key is replaced with an `{{ $env.GEMINI_API_KEY }}` expression — no real key is committed.
- Personal email addresses are replaced with `your-team-inbox@example.com`.
- SMTP credentials are referenced by n8n credential ID only; the actual password is never stored in the workflow file (n8n encrypts credentials server-side, separate from the workflow JSON).

If you fork or modify this workflow, **double-check before committing** that no real API keys, passwords, or personal email addresses have been reintroduced into the JSON. A leaked Gemini key gets scraped from public GitHub repos within minutes and can run up your bill.

## Lessons learned while building this

See [`docs/CHALLENGES.md`](docs/CHALLENGES.md) for the specific errors hit (deprecated model names, truncated AI output, SMTP auth failures, and more) and how each was resolved.

## Built with

- [n8n](https://n8n.io/) — workflow automation, selfhosted:- http://localhost:5678/workflow/FJeBG5XeUg6hhxkW
- [Google Gemini 2.5 Flash](https://ai.google.dev/) — email classification
- Gmail SMTP — email delivery
- n8n Webhook — ticket intake


