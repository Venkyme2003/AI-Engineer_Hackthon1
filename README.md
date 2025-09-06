# AI-Engineer_Hackthon1

Heads-up on the demo results (from your CSV): 19 support-type emails detected; 10 urgent, 9 not urgent; sentiment split: 12 negative, 7 neutral.

1) Architecture (4-day hackathon friendly)

High-level

Ingestion: Worker pulls emails via IMAP/Gmail API/Outlook Graph → stores raw messages in DB.

Classifier & Extractor: Fast pipeline tags Sentiment + Urgency and extracts contacts, requirements, product mentions.

RAG + Reply: LLM drafts context-aware replies using a lightweight knowledge base (FAQs, policies, troubleshooting playbooks).

Queue: Priority queue ensures Urgent first; agents can approve/edit/send.

Dashboard: React UI to review email, extracted data, draft reply, and analytics (24-hour stats, live filters).

Audit/Actions: One-click send, snooze, escalate, assign owner, mark resolved; store all states and timestamps.

Suggested stack

Backend: FastAPI (Python), Celery/RQ for background jobs, SQLite (hackathon) → PostgreSQL (prod).

NLP: OpenAI (GPT-4o mini/GPT-4.1) or local (DistilBERT/T5) for classification & drafting. Fallback rules for offline.

RAG: Chroma/FAISS vector store populated from Markdown/URLs/Docs; retrieval via cosine similarity + structured prompts.

Email: Gmail REST (OAuth) or standard IMAP (imaplib/IMAPClient) + SMTP for sending; Outlook Graph if needed.

Frontend: React + Tailwind + shadcn/ui; Recharts for graphs.

Auth: Google OAuth or email+OTP (since this is a demo tool).

2) Data model & flow

Tables

emails(id, provider_id, from_addr, subject, body, received_at, raw_json)

email_meta(email_id, sentiment, priority, extracted_emails, extracted_phones, products, requirements, kb_hits)

email_states(email_id, status ENUM('new','queued','drafted','sent','snoozed','escalated','resolved'), assignee, updated_at)

outbox(email_id, draft, final, sent_at)

kb_docs(id, title, content, tags, embedding)

events(id, email_id, type, payload_json, created_at) (audit log)

Processing

Fetch new messages → filter by subject keywords (“support”, “query”, “help”, “request”, etc.).

Classify sentiment (Positive/Negative/Neutral) + Priority (Urgent/Not urgent via keywords like “immediately/critical/cannot access” + model score).

Extract contacts (regex), products (n-grams + product dictionary), requirements (verb-led sentences).

RAG: Embed email; retrieve top-k KB chunks; build a structured prompt with:

Customer’s problem summary

Extracted facts (account, invoice, product, error)

Tone guardrails (empathetic if Negative)

Company policy snippets

Draft reply; enqueue urgent first; expose to UI for review; approve → send via SMTP/Gmail.

Update states & analytics.

3) Scoring logic (simple & robust)

Urgency score (0–1): keyword features (cannot access, down, critical, asap, immediately, refund now) + temporal penalty (older gets higher priority if untouched). Threshold → Urgent.

Sentiment: model + lexicon ensemble; if disagree, lean negative for safety (to trigger empathy).

Queue order: (Urgent desc) → (SLA time left asc) → (received_at asc).

4) Prompts (effective + short)

System

You are a customer support assistant. Write concise, professional, and empathetic replies. Use the provided company policies as the single source of truth.

RAG template

Email summary (auto-generated)

Extracted facts: {contacts, products, requirements}

Relevant policy chunks (top-k)

Constraints:

Acknowledge frustration if sentiment=Negative.

Reference product by name if found.

Ask only for missing essentials (invoice ID, error screenshot, timestamp/timezone, last 4 digits, etc.).

Offer next action + expected timeline.

Keep to ≤ 8 sentences.

5) Dashboard (what to show)

Left panel: Filters (Urgent toggle, Sentiment chips, Date range, Assignee).

Center list: Emails (sender, subject, age, badges for Urgent/Negative).

Right pane:

Raw email + Extracted (contacts, products, requirements)

Draft reply (editable); buttons: Approve & Send, Snooze, Escalate

Top stats: Total last 24h, Resolved, Pending, Sentiment mix, Urgent count; small bar chart + sparkline.

6) Code you can drop into a repo
Backend (FastAPI sketch)

/auth/login (OAuth)

/emails/ingest (background pull)

/emails (list with filters, pagination)

/emails/{id} (details + extracted)

/emails/{id}/draft (LLM draft + RAG)

/emails/{id}/send (SMTP/Gmail send + persist sent copy)

/analytics/summary?window=24h

Core Python modules

ingestor.py (IMAP/Gmail pull)

nlp.py (sentiment, urgency, extraction)

rag.py (embed, retrieve, assemble prompt)

drafts.py (LLM call + tone rules)

db.py (SQLModel / SQLAlchemy)

server.py (FastAPI routes)

worker.py (RQ/Celery jobs: fetch, classify, draft, send)

Frontend (React)

pages/Dashboard.tsx (list + filters + right pane)

components/EmailCard.tsx, DetailsPanel.tsx, ReplyEditor.tsx

pages/Analytics.tsx (cards + charts)

lib/api.ts (fetchers; React Query)

If you want, I can drop a single-file React dashboard scaffold into a Canvas so you can preview it live and copy-paste into your project.

7) What I already ran for you (local prototype)

Loaded your CSV → filtered support emails.

Classified Sentiment and Urgency (rule-based for hackathon speed).

Extracted emails/phones/products/requirements.

Mini-RAG over a tiny KB; generated draft replies with empathy when Negative.

Sorted a priority queue (Urgent first) and produced charts + export files.
Open the displayed table to inspect every draft reply and extracted metadata.

8) Submission package checklist (you can reuse this)

Demo video (3–5 min)

Show dashboard (filters, email list, detail, draft edit, send).

Show analytics (24h counts, urgent, sentiment chart).

Show live ingestion (mock mailbox or replay CSV).

Prove priority queue (urgent jumps to top).

Prove empathy (negative email → apologetic tone).

Docs (short, self-written)

Problem → Architecture diagram → Data model → Pipelines → Prompt examples → RAG source list → Security notes → Known limitations → Next steps.

Include quickstart:

backend: uvicorn server:app --reload
worker: rq worker supportq
frontend: npm run dev
env: OPENAI_API_KEY=..., IMAP_*, SMTP_*


SLA & safeguards: cap token usage, redact secrets (PII), manual approval for high-risk replies.

9) Stretch goals (if time permits)

SLA timers & breach alerts.

Auto-triage to teams (Billing, Access, Tech).

Multi-language detection + translation.

Duplicate detection (threading).

Feedback loop: “Was this helpful?” → fine-tune classifiers.

StatusPage + incident integration for outages.
