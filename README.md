# Speed to Lead & Lead Scoring System

An end-to-end inbound lead qualification and engagement system built on **n8n** that instantly scores, researches, and responds to leads via **Email**, **SMS**, and **Phone** — so your sales team only talks to prospects worth their time.

> Leads convert **391% more** when first contact is made within 5 minutes. This system makes first contact in **seconds**.

---

## The Problem

You're getting inbound leads but:

- You're **manually reviewing** every single one
- Hours are wasted qualifying junk leads
- By the time you reach out to the good ones, **they've gone cold**
- Follow-up is slow, inconsistent, and doesn't scale

---

## The Solution

Three n8n workflows that work together as a complete lead lifecycle engine:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        SPEED TO LEAD SYSTEM                             │
│                                                                         │
│  ┌───────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐ │
│  │   Lead     │───▶│  Score &     │───▶│  Multi-Chan  │───▶│  Sales   │ │
│  │   Form     │    │  Research    │    │  Engagement   │    │  Handoff │ │
│  └───────────┘    └──────────────┘    └──────────────┘    └──────────┘ │
│                          │                    │                         │
│                    ┌─────┴─────┐      ┌──────┴───────┐                 │
│                    │  Tavily   │      │  Email │ SMS  │                 │
│                    │  OpenAI   │      │  Phone        │                 │
│                    │  Airtable │      │               │                 │
│                    └───────────┘      └──────────────┘                 │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## System Architecture

The system consists of **3 interconnected workflows** totaling **153 nodes**:

### Workflow 1 — Speed to Lead & Lead Scoring (Main)

The orchestration layer. Triggers on form submission, scores leads, researches companies, and dispatches engagement via the appropriate channel.

```
                         ┌─────────────┐
                         │   Webhook    │
                         │  (Lead Form) │
                         └──────┬───────┘
                                │
                         ┌──────▼───────┐
                         │  Edit Fields  │
                         │  & Normalize  │
                         └──────┬───────┘
                                │
                    ┌───────────▼───────────┐
                    │  HubSpot: Search for  │
                    │  Existing Contact     │
                    └───────────┬───────────┘
                                │
                     ┌──────────▼──────────┐
                     │  New or Existing?    │
                     └────┬──────────┬─────┘
                          │          │
                   ┌──────▼──┐  ┌───▼────────┐
                   │ Create  │  │   Update    │
                   │ Contact │  │   Contact   │
                   └────┬────┘  └──────┬──────┘
                        └──────┬───────┘
                               │
                   ┌───────────▼────────────┐
                   │  AI Lead Scoring &     │
                   │  Evaluation Agent      │
                   │                        │
                   │  Tools:                │
                   │  • lead_scoring_criteria│
                   │  • Tavily Web Search   │
                   │  • Google Sheets       │
                   └───────────┬────────────┘
                               │
                   ┌───────────▼────────────┐
                   │  Channel Router        │
                   │  (Switch Node)         │
                   └────┬──────┬───────┬────┘
                        │      │       │
                ┌───────▼┐ ┌──▼────┐ ┌▼───────┐
                │ Email  │ │  SMS  │ │ Phone  │
                │ Agent  │ │ Agent │ │ Agent  │
                └───┬────┘ └──┬───┘ └──┬─────┘
                    │         │        │
                ┌───▼────┐ ┌──▼───┐ ┌──▼──────┐
                │ Gmail  │ │Twilio│ |  Retell │
                │ Send   │ │ Send │ │  Call   │
                └───┬────┘ └──┬───┘ └──┬─────┘
                    └─────────┼────────┘
                              │
                   ┌──────────▼────────────┐
                   │  Supabase: Log Every  │
                   │  Step for Audit Trail │
                   └───────────────────────┘
```

### Workflow 2 — Email Auto Responder

Handles ongoing email conversations after the initial outreach. Uses RAG (Retrieval Augmented Generation) with a Supabase vector store to answer prospect questions accurately.

```
  ┌──────────────┐    ┌─────────────┐    ┌─────────────────┐
  │ Gmail        │───▶│ Thread &    │───▶│ Match to Lead   │
  │ Trigger      │    │ Message     │    │ in HubSpot      │
  └──────────────┘    │ Retrieval   │    └────────┬────────┘
                      └─────────────┘             │
                                          ┌───────▼────────┐
                                          │   AI Agent     │
                                          │                │
                                          │  Tools:        │
                                          │  • Vector DB   │
                                          │    (Supabase)  │
                                          │  • Lead Q's    │
                                          │  • Scoring     │
                                          └───────┬────────┘
                                                  │
                                       ┌──────────▼──────────┐
                                       │  Status = "Open"?   │
                                       └────┬──────────┬─────┘
                                            │          │
                                     ┌──────▼──┐  ┌───▼──────────┐
                                     │ Continue │  │ Re-Score &   │
                                     │ Convo    │  │ Assign Rep   │
                                     └─────────┘  └───┬──────────┘
                                                      │
                                               ┌──────▼───────┐
                                               │ Notify Sales │
                                               │ Rep via Email│
                                               └──────────────┘
```

### Workflow 3 — SMS Auto Responder

Handles ongoing SMS conversations via Twilio. Includes a polling mechanism to batch rapid-fire texts before responding.

```
  ┌──────────────┐    ┌─────────────┐    ┌───────────────────┐
  │ Twilio       │───▶│ Store in    │───▶│ 3-Min Polling     │
  │ Trigger      │    │ Supabase    │    │ (Batch Messages)  │
  └──────────────┘    └─────────────┘    └─────────┬─────────┘
                                                   │
  ┌──────────────┐                        ┌────────▼─────────┐
  │ Schedule     │───▶ Postgres ─────────▶│ Match & Format   │
  │ Trigger      │    (Get Pending)       │ Messages         │
  └──────────────┘                        └────────┬─────────┘
                                                   │
                                          ┌────────▼─────────┐
                                          │   AI Agent       │
                                          │                  │
                                          │  Tools:          │
                                          │  • Vector DB     │
                                          │    (Supabase)    │
                                          │  • Lead Q's      │
                                          │  • Scoring       │
                                          └────────┬─────────┘
                                                   │
                                        ┌──────────▼──────────┐
                                        │  Status = "Open"?   │
                                        └────┬──────────┬─────┘
                                             │          │
                                      ┌──────▼──┐  ┌───▼──────────┐
                                      │ Send    │  │ Re-Score &   │
                                      │ SMS     │  │ Assign Rep   │
                                      └─────────┘  └──────────────┘
```

---

## How It Works (Step by Step)

### Phase 1: Lead Capture & Scoring

1. **Lead submits a form** via webhook (`POST /submit-lead-form`)
2. Fields are normalized and the lead is created or updated in **HubSpot**
3. An **AI Scoring Agent** evaluates the lead against predefined criteria stored in **Airtable / Google Sheets**
4. **Tavily** conducts web research on the company (if a company name or URL is provided)
5. **OpenAI** analyzes the score and research to determine qualification status

### Phase 2: Multi-Channel Engagement

Based on lead preferences, the system dispatches first contact:

| Channel | Tool | Response Time |
|---------|------|--------------|
| Email | Gmail + OpenAI | Seconds |
| SMS | Twilio + OpenAI | Seconds |
| Phone | Outbound Call API | Seconds |

### Phase 3: Automated Conversation

- **Email Responder** — Monitors Gmail for replies, retrieves thread context, uses RAG to answer questions, and continues the qualification conversation
- **SMS Responder** — Listens via Twilio trigger, batches rapid messages using a 3-minute polling interval in Postgres, then responds with full context
- Both channels ask qualifying questions from a curated question bank, never making up answers

### Phase 4: Scoring & Sales Handoff

When the conversation concludes (no more questions from the lead):

1. Lead status changes from `New` → `Open`
2. AI re-scores the lead based on the full conversation
3. A sales rep is assigned based on territory, expertise, or availability
4. The rep receives an email with the complete conversation history, score, and context
5. Everything is logged in **Supabase** for auditing

---

## Tech Stack

| Tool | Role |
|------|------|
| **n8n** | Workflow orchestration layer |
| **OpenAI (GPT)** | Lead scoring, email/SMS drafting, qualification decisions |
| **Tavily** | Web search tool for company research |
| **HubSpot** | CRM — stores lead info, scores, and research |
| **Airtable** | Stores scoring criteria, qualifying questions, sales rep info |
| **Google Sheets** | Backup data store (higher API limits than Airtable) |
| **Gmail** | Email send/receive for lead conversations |
| **Twilio** | SMS send/receive for text conversations |
| **Supabase** | Vector store (RAG), message logging, audit trail |
| **Postgres** | SMS message storage and batch polling |

---

## Files

| File | Description |
|------|-------------|
| `Speed_to_Lead_and_Lead_Scoring_System_Github.json` | Main orchestration workflow (50 nodes) |
| `response-email.json` | Email auto-responder workflow (52 nodes) |
| `response-text.json` | SMS auto-responder workflow (51 nodes) |

---

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- OpenAI API key
- Tavily API key
- HubSpot account with API/App token
- Airtable account with scoring criteria tables
- Gmail account with OAuth2 configured
- Twilio account with SMS-enabled phone number
- Supabase project with vector store and logging tables
- Postgres database for SMS message queuing

### Installation

1. **Import the workflows** — In n8n, go to *Workflows → Import from File* and import all three JSON files

2. **Configure credentials** — Set up credentials for each service in n8n's credential manager:
   - OpenAI API
   - HubSpot App Token
   - Airtable API Token
   - Gmail OAuth2
   - Twilio API
   - Supabase API
   - Postgres connection

3. **Set up Airtable / Google Sheets** — Create tables for:
   - Lead scoring criteria
   - Qualifying questions
   - Sales team directory

4. **Set up Supabase** — Create:
   - A `documents` vector store table (for RAG)
   - Logging tables for audit trail
   - Session tracking tables for SMS

5. **Configure the webhook** — Point your lead form to the webhook URL provided by n8n

6. **Activate all three workflows**

---

## Challenges & Solutions

### Airtable Rate Limits

Airtable's API call restrictions created bottlenecks at higher lead volumes. Critical scoring and business rule tables were replicated into **Google Sheets** (1,000+ API calls/month) to handle significantly more leads without hitting constraints.

### SMS Concurrency

When multiple leads texted simultaneously, messages would route incorrectly. This was solved with a **batch loop node** and custom matching formulas that ensure each message pairs with the correct lead record under high concurrent load.

### SMS Message Batching

Unlike email, people send multiple texts in rapid succession. A premature response to an incomplete thought creates a poor experience. The system stores all incoming messages in **Postgres** and polls every **3 minutes**, collecting the full thread before generating a response.

### Email Thread Matching

Concurrent email replies caused confused or misrouted responses. **Data processing formulas and filters** were added to accurately pair each incoming email with its corresponding lead thread.

---

## Future Roadmap

- **Multimedia support** — Handle images, videos, and attachments sent by leads via SMS and email
- **Sentiment analysis** — Detect urgency, frustration, or buying signals to escalate hot leads to human reps immediately
- **Score-to-revenue correlation** — Track which qualification scores predict actual deal closure and client value
- **Paid channel integration** — Feed validated scoring models into Google Ads and other platforms to build lookalike audiences based on high-converting lead profiles

---

## Demo

- [Lead Scoring & Qualification Walkthrough](https://www.loom.com/share/c7a259259ef142cea32bf5baf4a4ba29)
- [SMS Auto Responder Walkthrough](https://www.loom.com/share/bb2019860bb244038aa4fae39492b9ab)
- [Email Auto Responder Walkthrough](https://www.loom.com/share/dc83ecd42be64e798e36ff3afdf265ca)

---

## License

This project is provided as-is for educational and portfolio purposes. Note it was originally made for a company called Cartonix so you may have to change some aspects of it. 
