# Autonomous Canadian Immigration Lead Extraction & Qualification Pipeline

An event-driven, AI-powered automation workflow built in n8n designed to autonomously ingest, deduplicate, qualify, and route high-intent immigration leads from across the web (Reddit, Quora, YouTube) to help immigration consultancies scale operations.

---

## 🚀 System Architecture Overview

The system architecture manages the entire lifecycle of a lead—from discovery to routing—entirely within an asynchronous pipeline:

[Schedule Trigger] 
       │
       ├──> [Apify Scraper Engine] ──> [Split Out Node]
       │                                     │
       └──> [Google Sheets: Fetch Logs] ─────┼──> [Merge Node: Anti-Duplicate]
                                                   │
   ┌─────────────────── [Loop Over Items] <────────┘
   │                         │
   │                         ▼
   │                 [Basic LLM Chain] ──> [OpenAI / Groq Hybrid Setup]
   │                         │
   │                         ▼
   │               [Append Row to Sheet]
   │                         │
   │                         ▼
   │                     [If Node] ──(True: Score ≥ 80)──> [Gmail Hot Lead Alert]
   │                         │
   └─(Next Item in Batch)────┘

---

## 🛠️ Tech Stack & Technical Justification

* Orchestration Engine (n8n v1): Manages the core visual control loops, error catch behaviors (retryOnFail), and relational conditional logic branches seamlessly.
* Data Extraction Ingestion (Apify): Integrates the apify/google-search-scraper actor using complex Boolean string parameters to search across multiple social platforms simultaneously, maximizing resource efficiency.
* State Control & Deduplication (Google Sheets Integration): Queries current database entries simultaneously alongside the scraper, utilizing a dual-input Merge Node configured to keepNonMatches on the unique url key. This forms an anti-duplication gateway so old profiles are never double-processed.
* Dual-LLM Resiliency Infrastructure (LangChain + Groq + OpenAI): Features a high-speed open-source framework via Groq as the principal validator, backed up by OpenAI’s Chat Model (gpt-5-nano) to safeguard system uptime and data flow processing.

---

## 🧠 AI Qualification & Data Engineering Rules

The underlying AI parsing agent relies on explicit data parameters enforced via a Structured Output Parser to convert messy text rows into deterministic JSON.

### Vetting Categories Evaluated:
1. Platform Mapping: Resolves the exact source node (Reddit, Quora, or YouTube).
2. Context Validation: Separates real user questions from generic community pages. If a profile contains no explicit question, it is auto-marked as genuine: "No" and assigned a cold score.
3. Visa Classification: Maps the text to strict immigration verticals: Canada Visitor Visa, Canada Student Visa, Canada Work Permit, Canada PR, Canada Spousal Visa, or Canada Super Visa.
4. Intent & Urgency Metrics: Detects indicators of tight timelines, structural visa expirations, or recent application refusals to tier profiles into Hot, Warm, or Cold.
5. Algorithmic Scoring: Calculates a precise metric from 0 to 100 assessing conversion likelihood. 

### Hot Lead Routing Channel
When an item registers a score ≥ 80 (or structural Hot intent), the system triggers an automatic fork through an If Node, passing contextual properties straight to a Gmail Notification Node that drafts an immediate action email containing an AI-generated, highly personalized outreach pitch.

---

## ⚠️ Known Limitations & Mitigations

* High-Dependency Metadata: Web scraper indexing can occasionally hide raw usernames or explicit publication timelines from snippet preview boundaries.
  - Mitigation: Handled seamlessly via embedded JavaScript expressions inside the data mapping fields, setting default fallback structures and dynamic chronological timestamps.
* API Rate Thresholds: High concurrent loops run the risk of causing transactional exceptions during bulk requests.
  - Mitigation: Configured native retryOnFail rules alongside a robust waitBetweenTries: 5000 execution rule to guarantee self-healing execution logs.

---

## 🚀 Production Scaling Roadmap (Commercialization)

To step this infrastructure up into an enterprise-grade B2B SaaS platform for worldwide legal consultancies, the following upgrades are mapped:

* Multi-Tenant Relational Databases: Migrating data targets from Google Sheets into high-velocity engines like PostgreSQL or Supabase to manage secure client partition boundaries.
* Direct Webhook Streaming: Replacing search scrapers with native platform APIs (like the official Reddit Developer API and Apify’s specific Quora/YouTube comment scraping actors) to capture raw metadata and user activity in real-time.
* Omnichannel Delivery Integration: Funneling validated hot leads directly into popular systems like HubSpot or Salesforce, alongside multi-channel text platforms (WhatsApp, SMS, Telegram) to compress follow-up times to seconds.

---
*Developed as an engineering demonstration for the Savoir AI - OTFCoder Private Limited Evaluation Committee.*