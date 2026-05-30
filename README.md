# 🇨🇦 AI Lead Qualifier — Canadian Immigration Consultancy
### n8n Workflow: `AI_INTERN_TASK`

An automated, AI-powered lead generation and qualification pipeline that scrapes immigration-related discussions from Reddit, Quora, and YouTube — scores them using an LLM, deduplicates against existing records, logs qualified leads to Google Sheets, and instantly emails hot leads to your team.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Workflow Architecture](#workflow-architecture)
- [Node-by-Node Breakdown](#node-by-node-breakdown)
- [Lead Scoring & Classification](#lead-scoring--classification)
- [Output Schema](#output-schema)
- [Prerequisites & Setup](#prerequisites--setup)
- [Environment & Credentials](#environment--credentials)
- [Configuration](#configuration)
- [How to Run](#how-to-run)
- [Email Alert Format](#email-alert-format)
- [Troubleshooting](#troubleshooting)

---

## Overview

This workflow runs on a **schedule** and performs the following high-level steps:

1. **Scrapes** Google Search results for immigration-related posts on Reddit, Quora, and YouTube via the Apify Google Search Scraper actor.
2. **Deduplicates** scraped results against URLs already stored in Google Sheets.
3. **Qualifies** each new result using an LLM (OpenAI GPT-5 Nano or Groq fallback), extracting structured lead data.
4. **Appends** every qualified lead to a Google Sheet ("Reddit Immigration Leads").
5. **Alerts** your team by Gmail for any lead with `Intent = Hot` **or** `lead_score ≥ 80`.

---

## Workflow Architecture

```
Schedule Trigger ──┬──► Run Apify Actor ──► Split Out ──────────────────┐
                   │                                                      ▼
                   └──► Get rows (GSheet) ────────────────────► Merge (dedup by URL)
                                                                          │
                                                                          ▼
                                                                  Loop Over Items
                                                                          │
                                                                          ▼
                                              ┌── OpenAI GPT-5 Nano ──►  │
                                              └── Groq (fallback)    ──► Basic LLM Chain
                                                         ▲                │
                                              Structured Output Parser ◄──┘
                                                                          │
                                                                          ▼
                                                               Append row in GSheet
                                                                          │
                                                                          ▼
                                                              If (Intent=Hot OR Score≥80)
                                                               │                  │
                                                       [TRUE] ▼           [FALSE] ▼
                                                     Send Gmail Alert    Loop (next item)
                                                               │
                                                               └──────► Loop (next item)
```

---

## Node-by-Node Breakdown

### 1. 🕐 Schedule Trigger
**Type:** `n8n-nodes-base.scheduleTrigger`

Kicks off the entire workflow on a recurring schedule (configure interval in node settings). This ensures fresh leads are captured automatically without manual intervention.

---

### 2. 🕷️ Run an Actor and get dataset
**Type:** `@apify/n8n-nodes-apify.apify`
**Actor:** `apify/google-search-scraper` (ID: `nFJndFXA5zjCTuudP`)

Runs the Apify Google Search Scraper with the following query set:

```
Canada visitor visa OR student visa help reddit
Canada work permit OR PR timeline reddit
Canada spousal OR super visa quora
Canada student visa OR PR process youtube
```

**Settings:**
- `maxPagesPerQuery`: 1 (one page of Google results per query)

Returns a dataset of `organicResults` containing URLs, titles, descriptions, and website metadata.

---

### 3. ✂️ Split Out
**Type:** `n8n-nodes-base.splitOut`

Splits the `organicResults` array from the Apify dataset into individual items so each search result is processed independently downstream.

---

### 4. 📊 Get row(s) in sheet
**Type:** `n8n-nodes-base.googleSheets`
**Sheet:** `Reddit Immigration Leads` → `Sheet1`

Fetches all existing rows from the Google Sheet. This is used for **deduplication** — URLs already logged are excluded from processing.

---

### 5. 🔀 Merge (Deduplicate)
**Type:** `n8n-nodes-base.merge`
**Mode:** Combine → Keep Non-Matches (Left Anti Join)
**Merge Key:** `url`

Compares the freshly scraped URLs (Input 1) against those already stored in Google Sheets (Input 2). Only URLs **not** already in the sheet pass through — preventing duplicate leads.

---

### 6. 🔁 Loop Over Items
**Type:** `n8n-nodes-base.splitInBatches`

Iterates over each new, deduplicated search result one at a time, feeding them sequentially to the LLM qualification chain.

---

### 7. 🤖 Basic LLM Chain
**Type:** `@n8n/n8n-nodes-langchain.chainLlm`

The core AI qualification engine. Receives the following fields from each search result:
- `websiteTitle`
- `title`
- `description`
- `url`

Applies an expert prompt as an **AI Lead Qualifier for a Canadian Immigration Consultancy**, evaluating each result according to strict mapping rules (see [Lead Scoring & Classification](#lead-scoring--classification)).

Uses the **Structured Output Parser** to enforce a JSON schema on the response.

**Sub-nodes attached:**
- OpenAI Chat Model (primary)
- Groq Chat Model (fallback)
- Structured Output Parser

---

### 8. 🧠 OpenAI Chat Model
**Type:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
**Model:** `gpt-5-nano`

Primary LLM for lead classification. Connected as a sub-node to the Basic LLM Chain.

---

### 9. ⚡ Groq Chat Model
**Type:** `@n8n/n8n-nodes-langchain.lmChatGroq`
**Model:** `openai/gpt-oss-120b`

Fallback LLM connected to the same LLM Chain. Activates if the primary OpenAI model is unavailable or encounters an error.

---

### 10. 📐 Structured Output Parser
**Type:** `@n8n/n8n-nodes-langchain.outputParserStructured`

Enforces structured JSON output from the LLM. Guarantees each result conforms to the lead schema before being written to Google Sheets. (See [Output Schema](#output-schema).)

---

### 11. 📝 Append row in sheet
**Type:** `n8n-nodes-base.googleSheets`
**Operation:** Append
**Sheet:** `Reddit Immigration Leads` → `Sheet1`

Writes every qualified lead (including non-hot ones) to the Google Sheet with all extracted fields.

**Mapped Columns:**
`Genuine`, `Category`, `Intent`, `Urgency`, `Summary`, `Outreach`, `Platform`, `url`, `lead_score`

---

### 12. ❓ If (Hot Lead Filter)
**Type:** `n8n-nodes-base.if`

Routes leads based on urgency:

| Condition | Value | Action |
|-----------|-------|--------|
| `Intent` equals | `"Hot"` | → Send Gmail Alert |
| `lead_score` ≥ | `80` | → Send Gmail Alert |
| Neither | — | → Loop (next item) |

Both conditions use **OR logic** — either one triggers an alert.

---

### 13. 📧 Send a message
**Type:** `n8n-nodes-base.gmail`
**To:** `brijrpatel007@gmail.com`

Sends a formatted email alert for hot leads. Subject line dynamically includes the category and score:
```
[HOT LEAD] Student Visa | Score: 92/100
```

See [Email Alert Format](#email-alert-format) for the full template.

---

## Lead Scoring & Classification

The LLM evaluates each search result on the following dimensions:

| Field | Description | Possible Values |
|-------|-------------|-----------------|
| `Platform` | Source platform | `Reddit`, `Quora`, `YouTube`, `Other` |
| `genuine` | Is this a real user seeking immigration help? | `Yes`, `No`, `Maybe` |
| `category` | Immigration category of the post | `Student Visa`, `Work Permit`, `PR`, `Visitor Visa`, `Spousal Visa`, `Super Visa`, `Other` |
| `intent` | Lead buying intent / urgency | `Hot`, `Warm`, `Cold` |
| `urgency` | Time-sensitivity of their need | `High`, `Medium`, `Low` |
| `summary` | 1–2 line human-readable summary | Free text |
| `outreach` | Ready-to-use personalized pitch | Free text (2 sentences) |
| `url` | Source URL | URL string |
| `lead_score` | Composite quality score | `0` – `100` |

**Hot lead** = `Intent: Hot` OR `lead_score ≥ 80`

---

## Output Schema

```json
{
  "Platform": "Reddit",
  "genuine": "Yes",
  "category": "Student Visa",
  "intent": "Hot",
  "urgency": "High",
  "summary": "User asking about study permit timelines for September intake.",
  "outreach": "Hi there! Based on your question about study permit timelines, we can help you navigate the process quickly. Our consultants specialize in fast-tracked student visa applications.",
  "url": "https://www.reddit.com/r/ImmigrationCanada/comments/example/",
  "lead_score": 95
}
```

---

## Prerequisites & Setup

Before importing this workflow into n8n, ensure you have the following:

**Accounts & Access:**
- [n8n](https://n8n.io/) instance (self-hosted or cloud)
- [Apify](https://apify.com/) account with access to `apify/google-search-scraper`
- [OpenAI](https://platform.openai.com/) API key (GPT-5 Nano access)
- [Groq](https://console.groq.com/) API key (fallback LLM)
- Google account with access to Google Sheets and Gmail
- A Google Sheet named **"Reddit Immigration Leads"** with the column headers listed below

**Google Sheet Column Headers (Sheet1):**
```
Genuine | Category | Intent | Urgency | Summary | Outreach | Platform | url | lead_score
```

---

## Environment & Credentials

Set up the following credentials in your n8n instance:

| Credential | Node(s) | Notes |
|------------|---------|-------|
| Apify API Token | Run an Actor | Create at apify.com/account/integrations |
| OpenAI API Key | OpenAI Chat Model | Requires GPT-5 Nano access |
| Groq API Key | Groq Chat Model | Free tier available |
| Google OAuth2 | Get row(s) in sheet, Append row in sheet | Needs Sheets + Drive scopes |
| Gmail OAuth2 | Send a message | Needs Gmail send scope |

---

## Configuration

Key settings you may want to customize:

**Search Queries** (in the Apify node → `customBody`):
```json
{
  "queries": "Canada visitor visa OR student visa help reddit\nCanada work permit OR PR timeline reddit\n...",
  "maxPagesPerQuery": 1
}
```
Edit the `queries` string to target different keywords, platforms, or visa types.

**Schedule Interval:** Edit the Schedule Trigger node to set how often the workflow runs (e.g., daily, every 6 hours).

**Hot Lead Threshold:** In the `If` node, change `80` to adjust the minimum `lead_score` that triggers an email alert.

**Alert Recipient:** In the `Send a message` (Gmail) node, update the `sendTo` field to your team's email address.

**Google Sheet ID:** Both Google Sheets nodes reference document ID `1G2LhX8a6Q43s7Yhsk29saRGs2uufPVhQ6_0evUdl_Qs`. Update this to your own sheet's ID (found in the Google Sheets URL).

---

## How to Run

1. Import `AI_INTERN_TASK.json` into your n8n instance via **Workflows → Import from file**.
2. Open the workflow and configure all credentials (see above).
3. Update the Google Sheet ID and alert email in the relevant nodes.
4. Set your desired schedule in the **Schedule Trigger** node.
5. Click **Activate** to enable the workflow.
6. To test immediately, click **Execute Workflow** for a manual run.

---

## Email Alert Format

When a hot lead is detected, your team receives an email formatted like this:

```
Subject: [HOT LEAD] Student Visa | Score: 92/100

Hello Team,

Our AI Lead Qualifier has intercepted a high-priority immigration prospect
that requires immediate attention.

=========================================
🚨 LEAD INSIGHTS
=========================================
📌 Platform:     Reddit
🎯 Category:     Student Visa
🔥 Intent:       Hot
⚡ Urgency:      High
📊 Lead Score:   92/100

📋 CASE SUMMARY:
User is asking about September intake study permit timelines and processing times.

=========================================
💬 READY-TO-USE OUTREACH PITCH
=========================================
Hi there! Based on your question about study permit timelines, we specialize
in fast-tracking student visa applications. Book a free consultation today.

=========================================
🔗 ACTION REQUIRED
=========================================
[Link to original post]
```

---

## Troubleshooting

**Workflow runs but no leads appear in Google Sheets:**
- Verify the Apify actor ran successfully and returned `organicResults`.
- Check that the Google Sheets credentials have write permissions.
- Confirm the sheet column headers exactly match the mapped field names.

**LLM returns malformed JSON:**
- The Structured Output Parser enforces the schema — if it fails, the item is skipped. Check n8n execution logs for parsing errors.
- Try switching to the Groq fallback model if OpenAI responses are inconsistent.

**Duplicate leads appearing:**
- Ensure the `Get row(s) in sheet` node is reading the correct sheet and document ID.
- The Merge node deduplicates by `url` — verify the `url` column is populated correctly in your sheet.

**No email alerts being sent:**
- Check that the `If` node conditions are correctly set (`Intent = Hot` OR `lead_score ≥ 80`).
- Verify Gmail OAuth2 credentials are active and the `sendTo` address is correct.

---

*Built with [n8n](https://n8n.io/) · Powered by Apify, OpenAI & Groq*
