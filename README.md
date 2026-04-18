# Automation System — Full Spec (Steps, Tech, Privacy)

This document defines **what to build**, **in what order**, and **how to talk about privacy** with clients or vendors. It assumes: **the AI brain proposes** (plans, drafts, classifies, researches); **your backend orchestrates**; **workers execute** — never the reverse.

You can implement the brain as **raw Claude API** or **Dify** (recommended when you want one productized layer for many business operations). Under Dify you still typically use **Claude** (or another provider) as the **model** inside apps/workflows.

---

## 0. High-Level Architecture (What You’re Actually Building)

You are building:

**Command → AI Brain → Orchestrator → Executors**

```
Mobile UI → API Gateway → AI Brain (Dify and/or Claude) → Task Orchestrator → Workers → External APIs
```

- **AI brain** turns natural language into **structured plans**, **drafts** (email reply, outreach copy), **research summaries** (influencers, sponsors), and **labels** (intent, priority) — **not** trusted to perform side effects alone.
- **Orchestrator** validates, queues, retries, and tracks state; enforces allowlists and human approval for money.
- **Workers** perform side effects (send email, browser automation, PayPal after approval, screenshots, storage).

### 0.1 Why add **Dify** as the AI brain

**Dify** ([dify.ai](https://dify.ai)) is an LLM application platform: **Chatflow / Workflow apps**, **Knowledge** (RAG over your docs), **Tools** (HTTP to your API), **Agents**, tracing, and versioning — with **Claude** (or GPT, etc.) as the model behind the scenes.

Use it when you want **one place** to grow many operations without rewriting prompts and glue for each:

| Business area | What Dify is good at | What still belongs in **your** orchestrator + workers |
|---------------|----------------------|--------------------------------------------------------|
| **Influencer / sponsor discovery** | Multi-step reasoning, web search tool (if enabled), summarizing lists, ranking criteria | Persisting results, rate limits, crawling policy, CRM export, scheduled jobs |
| **Email reply** | Drafting replies from thread context + brand voice (RAG) | Actually **sending** via Gmail API, threading IDs, opt-out, quotas |
| **Email send / sequences** | Subject/body variants, personalization text | **Send** worker, bounce handling, unsubscribe, domain reputation |
| **Inbound triage** | Classify “sponsor vs spam vs support”, extract entities | Route to queues, auto-task creation |
| **Full business ops** | Many apps: one app per workflow family, shared knowledge base | Single **orchestrator** contract: Dify returns **JSON** or hits your **tool** endpoints; workers stay the source of truth for execution |

**Deployment note:** Dify can be **self-hosted** (your VPC) or **cloud**; self-hosting aligns better with “data stays in my environment” if that is a hard requirement.

---

## 1. Recommended Tech Stack

| Layer | Choice | Role |
|--------|--------|------|
| **Core backend** | Node.js (**NestJS** for structure, or **Express** for minimal surface) | APIs, auth, orchestration |
| **Database** | **PostgreSQL** | Commands, tasks, transactions, audit logs |
| **Queue / jobs** | **Redis** + **BullMQ** | Reliable async execution, retries |
| **Elite reliability (optional later)** | **Temporal** | Long-running workflows, stronger guarantees than a queue alone |
| **AI brain (pick one or combine)** | **Dify** (recommended for breadth) **and/or** **Anthropic Claude API** | Planning, drafting, RAG, multi-app workflows; model can still be Claude inside Dify |
| **Quick integrations (optional)** | **Make.com** | Prototypes or non-critical glue — **not** where core business logic lives |
| **Object storage** | **S3** or **GCS** | Screenshots, exports, attachments |
| **Auth to third parties** | **OAuth** (Google for Gmail, PayPal for payouts after approval) | Scoped tokens, no password storage |

---

## 2. System Components

### A. Command gateway (mobile entry point)

Options (pick one to start, combine later):

- **PWA in the browser** (recommended): one URL, works on iPhone/iPad Safari and Chrome.
- **Telegram bot**: very fast MVP for “type command, get status.”
- **REST**: `POST /command` with body like:

```json
{
  "input": "Send emails to 10 AI startups"
}
```

### B. AI brain — **Dify** and/or **Claude**

**Role:** parse intent, **explore** domains (e.g. influencer fit, sponsor angles), **draft** email replies and campaigns, decompose into **typed steps**, attach **citations** when using Knowledge — still **no direct execution** of money, mass email, or unchecked sends.

#### Option 1 — **Claude API only** (minimal stack)

Your backend calls Messages API with a strict planner prompt; you validate JSON and enqueue tasks.

#### Option 2 — **Dify as the brain** (scale-friendly)

- **One Dify “app” per family** of operations (e.g. `influencer_research`, `email_reply_draft`, `sponsor_outreach_plan`, `inbox_triage`) or **one Workflow** with routing.
- **Knowledge**: paste brand guidelines, product one-pagers, past winning emails — RAG for consistent tone.
- **Tools (HTTP)**: Dify calls **your** API for “create task,” “list pipeline,” “fetch last thread” — your server never exposes raw secrets to Dify; pass **opaque tokens** or **server-side session** only.
- **Output contract**: configure the app/workflow to end with **JSON** matching your orchestrator schema (same as below), or use a **Code** / structured output node so the model is forced into a machine-readable shape.

Example structured output the orchestrator expects (illustrative — same whether the model ran in Dify or raw Claude):

```json
{
  "workflow": "email_outreach",
  "steps": [
    { "type": "find_targets", "query": "AI startups" },
    { "type": "send_email", "count": 10 }
  ]
}
```

**Integration pattern (recommended):**

```
Mobile → Your API → Dify (Chat / Workflow API) → structured JSON + optional tool calls
                → Orchestrator validates → BullMQ → Workers (Gmail, Playwright, PayPal gated, …)
```

**Rule:** Even with Dify **Agent** mode, **do not** give the agent unrestricted keys to PayPal or SMTP. Use **tools** that map to **orchestrator endpoints** that enforce allowlists and approvals.

### C. Task orchestrator (core system)

**Rule:** Do **not** let the LLM (Claude or Dify-driven agent) **bypass** validation or execute side effects without your allowlist and queues.

**Orchestrator responsibilities:**

- Validate task types and parameters against an allowlist.
- Create `tasks` rows, enqueue jobs.
- Retries with backoff; idempotency keys where needed.
- Status transitions: `pending` → `running` → `completed` / `failed`.
- Human gates for money movement (see PayPal).

### D. Worker services (execution layer)

Split by domain (can be processes or services):

| Worker | Responsibility | Typical stack |
|--------|----------------|----------------|
| **Email** | Gmail API (or provider), send, log bounces/errors | OAuth + provider SDK |
| **Finder / scraper** | Companies, influencers, sponsors | Official APIs first; **Playwright** only when necessary |
| **Application / forms** | Autofill, submissions | Playwright with strict selectors + human review queue for fragile sites |
| **PayPal** | **Never blind automation** | AI brain **prepares** → **pending transaction** → **mobile one-tap approve** → worker executes PayPal API |
| **Screenshot** | Receipt / confirmation pages | Playwright or Puppeteer → upload to S3/GCS → link in DB |

---

## 3. Data Model (Simplified)

Conceptual tables:

**`commands`**

- `id`, `user_id`, `input`, `parsed_output` (JSON), `status`, timestamps

**`tasks`**

- `id`, `command_id`, `type`, `payload` (JSON), `status`, `attempts`, `last_error`, timestamps

**`transactions`** (money)

- `id`, `user_id`, `amount`, `receiver`, `metadata`, `status` (`pending` | `approved` | `sent` | `failed`), `external_ref`

**`files`**

- `id`, `user_id`, `url` or `storage_key`, `type` (`screenshot`, `export`, …), `related_task_id`

**`ingestion_events`** (recommended for webhooks)

- `id`, `provider`, `external_event_id` (**unique** with `provider`), raw payload ref, `processed_at`, `command_id` (optional)

Add indexes on `command_id`, `user_id`, `status`, and created time for dashboards.

---

## 4. Full Execution Flow (Example)

**Command:** “Send emails to 5 fitness influencers”

1. User submits command from mobile UI → API persists `commands` row.
2. Backend calls **Dify** (workflow/chat API) and/or **Claude** with a **planner-only** contract → receives **validated** JSON plan (same schema either way).
3. Orchestrator validates steps → creates `tasks` → enqueues BullMQ jobs.
4. Workers run (e.g. finder, then email) → append logs, update status.
5. UI polls or uses SSE/WebSocket for status; final summary returned.

**PayPal + screenshot variant:** after “prepare,” create `transactions` as `pending` → push notification or in-app banner → user approves once → worker sends via PayPal API → screenshot worker captures receipt → file row + folder prefix in storage.

---

## 5. Security & Privacy Architecture

### Technical truth (for your own design)

- The **automation platform** will hold **OAuth tokens** and encrypted secrets **for your account** — that is how Gmail, PayPal, and browsers-on-your-behalf work.
- **Best practice:** scoped OAuth, short-lived tokens where possible, **AES-256 envelope encryption** at rest, separate secrets/config service, no raw passwords, no credentials in repos.

### Client-facing privacy language (English, final)

Use this when scoping work with a freelancer or agency:

> **Privacy & security**  
> All data, connected accounts, and credentials belong to you and are used only to run automations you explicitly trigger. Access is enforced with authentication, least-privilege OAuth scopes, and encryption at rest for stored tokens. The implementer’s role is **technical setup and configuration** of **your** infrastructure or **your** cloud project; they should **not** retain copies of your passwords or long-lived secrets outside the agreed secure channels (e.g. your vault or your deployment secrets). **Human approval** is required for sensitive financial actions (e.g. PayPal sends).

### What you should **never** promise literally

Avoid: “Claude executes everything with no system in the middle” or “nobody can ever access tokens” — tokens exist on your servers; security is **how** they are stored, scoped, rotated, and who can deploy code.

---

## 6. Mobile Interface — Next.js vs React

**Recommendation: Next.js (App Router) for this product.**

| Criterion | Next.js | Plain React (e.g. Vite + SPA) |
|-----------|---------|----------------------------------|
| **API routes / BFF** | Built-in `app/api` — easy to hide API keys, one deploy | You add a separate backend or proxy anyway |
| **PWA on iPhone** | Same as React; add manifest + service worker | Same |
| **Auth, SSR, metadata** | First-class | More manual setup |
| **“Simple text box + button”** | Trivial in either | Trivial in either |

**Bottom line:** For “mobile browser + command + status + approvals,” **Next.js + Tailwind** is the best default: one codebase, API next to UI, straightforward hosting (Vercel, Fly.io, VPS).

**Minimal UX (your stated requirement):**

- One text area for the command.
- One primary button: **Run** / **Submit**.
- Result area: success message, task list, or error — readable on Safari/Chrome on iPhone/iPad.

---

## 7. AI Configuration — Dify + Claude (Critical)

Whether prompts live in **Dify** or in your code calling **Claude** directly, use the same **non-negotiables**:

- **Planner / operator split:** the model proposes **plans and drafts**; it does **not** “confirm money sent” or bypass your queue.
- **Structured output:** final step returns **JSON only** (or a dedicated parser node in Dify) matching your **JSON Schema** / Zod types.
- **Allowlisted `task.type` values** only (e.g. `email_outreach`, `influencer_research`, `email_reply_draft`, `sponsor_search`, `paypal_prepare`, `screenshot_receipt`).
- **No invented credentials**; no promises that actions already happened unless a **worker** recorded success.
- **Ambiguity:** return `clarifying_questions` instead of guessing.

**Dify-specific tips:**

- Use **Knowledge** for brand, product, and compliance snippets so influencer/sponsor/email work stays on-message.
- Use **Workflow** for deterministic branches (e.g. “if user asked for reply only → emit `email_reply_draft` tasks only”).
- Wire **Tools** to your backend for anything that touches **private** data or **side effects**; keep Dify API keys server-side (Next.js API route or NestJS), never in the mobile client.

Always run the final payload through **schema validation** before the orchestrator enqueues work.

---

## 8. Logging & Observability

- Structured logs (**Pino** or **Winston**): `command_id`, `task_id`, `worker`, `duration_ms`, `error_code`.
- Store per-task log lines in DB or log drain for search.
- Optional: Datadog / Grafana Loki later.

---

## 9. Deployment (Pragmatic Tiers)

**Simple (start here):**

- Backend on a **VPS** or PaaS.
- Managed **PostgreSQL** + managed **Redis**.
- Object storage **S3/GCS**.

**Advanced:**

- Kubernetes, one deployment per worker type, autoscaling from queue depth.

---

## 10. What Will Break (Design For It)

| Risk | Mitigation |
|------|------------|
| Scraping / DOM changes | Prefer APIs; version selectors; human review queue |
| PayPal policy / limits | Official APIs; pending + user approval; audit log |
| Email deliverability | Rate limits, domain auth (SPF/DKIM), opt-in lists |
| Model hallucinations | Allowlist task types, validation, refusal paths |

---

## 11. Documentation Structure (For the Project Repo)

1. Overview  
2. Architecture diagram  
3. Services & responsibilities  
4. API endpoints  
5. AI prompt design + JSON schema (Dify apps / Claude)  
6. Worker definitions  
7. Security model  
8. Deployment guide  
9. Failure handling & runbooks  

---

## 12. Build Order (Do Not Skip)

1. **Command ingest** + persist + auth.  
2. **AI brain → JSON plan**: stand up **Dify** (or Claude API) with one **golden schema** + strict validation.  
3. **One real worker** (email is a good first win).  
4. **Queue + retries + task UI**.  
5. **Approval flow** (generic first, then PayPal-specific).  
6. **PayPal prepare → approve → execute**.  
7. **Screenshot + organized storage paths** (e.g. `userId/yyyy/mm/dd/txnId.png`).  
8. **Scraping / application automation** last (highest maintenance).

---

## 13. English Templates You Can Send

### A. Full request to a vendor (includes privacy)

Hi,

I need an automation system where I type a **command from my phone** and the system **runs the workflow** I approve. The **AI layer** should **plan and draft** (including influencer/sponsor research, email replies, and outreach copy) using **Dify** with **Claude** (or equivalent) as the model, outputting **structured JSON**; a **backend orchestrator** must **validate and execute** via workers (not unchecked direct model execution).

**Tasks the system should support:**

- Send emails to companies and influencers  
- Apply to startup and investment platforms (with a review/queue step where needed)  
- Find and contact sponsors for my app  
- Handle **daily prize transfers via PayPal** — I **approve each transfer with one click**; the system **prepares** everything beforehand  
- Automatically **screenshot** each PayPal transfer confirmation  
- Save screenshots to an **organized folder** in cloud storage  

**Tools:**

- **Dify** as the AI application layer (workflows, knowledge, tools); **Claude** (or another approved LLM) inside Dify for reasoning and writing  
- **Make.com** is optional for quick integrations, **not** for core money or compliance logic  

**Interface:**

- **Mobile-friendly web app** (Safari/Chrome on iPhone/iPad): command box, one execute button, on-screen result/status  

**Privacy & security:**

- All my data, accounts, and credentials are **private** and intended for **my use only**  
- OAuth, scoped permissions, encrypted token storage; **no raw password storage**  
- Your role is **technical setup and configuration**; do **not** ask me to share credentials in chat — use secure secret channels / my cloud project  

Please share **cost**, **timeline**, and **assumptions** (hosting, accounts I must create, ongoing maintenance).

Thank you.

### B. Short clarification — simple mobile UI (English)

Yes. I need a **simple mobile-friendly web interface** where I can:

- Type my command in a **text box**  
- Press **one button** to run it  
- See the **result or confirmation** on the same screen  

It must work in **Safari or Chrome on iPhone/iPad**. Please keep the UI as **minimal** as possible.

**Stack preference:** **Next.js** (with Tailwind) for the web app, with server-side API routes so keys stay off the device.

---

## 14. Optional Next Step (Internal)

If you want implementation scaffolding next, ask for: **“NestJS + BullMQ + Next.js PWA folder layout”**, **“Dify workflow + tool endpoints contract”**, **“webhook + cron trigger handlers + idempotency schema”**, or **“SQL migrations for commands/tasks/transactions/files.”**

---

## 15. Dify ↔ Full business operations (mental map)

| User-facing intent | Typical Dify responsibility | Your platform responsibility |
|--------------------|-----------------------------|------------------------------|
| “Find 20 fitness influencers in GCC” | Research steps, criteria scoring, table in reply | Store leads, dedupe, respect ToS/APIs, optional scheduled refresh |
| “Reply to this sponsor email professionally” | Draft from Knowledge + thread summary | Fetch thread via worker, show draft in UI, **send** only after user confirms |
| “Send intro emails to this list” | Personalize bodies, subject lines | Throttling, unsubscribe, provider API, logs |
| “Prepare today’s PayPal prizes” | Wording, recipient list structuring, amounts check | `pending` transactions, one-tap approve, audit trail |
| “What broke last week?” | Summarize logs you pass in (or RAG over runbooks) | Source of truth: DB + log store |

This keeps **Dify** as the **exploration and language brain** and your stack as the **system of record and execution**.

---

## 16. Best workflows & triggers (design guide)

Good automation is **event-driven**, **idempotent**, and **split**: **Dify** handles *language + structure*; **your backend** handles *triggers, durability, and execution*.

### 16.1 Two layers of “workflow” (do not merge them blindly)

| Layer | Owns | Should not own |
|-------|------|------------------|
| **Dify Workflow / Chatflow** | Prompts, RAG, branching questions, drafts, JSON plan output, light HTTP tools to *your* API | Long retries, money movement, bulk email sends, secrets, final “sent” truth |
| **Platform workflow** (orchestrator + DB + queue) | Triggers, scheduling, approvals, task state, retries, DLQ, audit log, webhooks verification | Open-ended chat without a contract |

**Rule:** Every Dify run that can create work should end with a **machine contract**: versioned **JSON schema** your orchestrator accepts (or a tool call that creates a `command` row — still validated server-side).

### 16.2 Trigger taxonomy (use the right front door)

| Trigger | When to use | Implementation sketch |
|---------|-------------|-------------------------|
| **User command** | “Run this now” from mobile / API | `POST /command` → persist → call Dify → validate JSON → enqueue |
| **Schedule (cron)** | Daily PayPal prep, digest emails, weekly sponsor scan | Managed cron / BullMQ **repeatable** job → create `command` with `trigger: cron` + frozen payload |
| **Inbound webhook** | New Gmail thread, Stripe event, Typeform, sponsor form | `POST /webhooks/:provider` → **verify signature** → normalize → **enqueue** (return `202` fast; do not run LLM inline if it risks timeout) |
| **Queue / task completion** | Step 2 after step 1 succeeds | Worker finishes → orchestrator emits next job or calls Dify again with **summarized** context |
| **Human approval** | Send email, PayPal payout, risky scrape | UI action `POST /approvals/:id/confirm` → flips DB state → **that** enqueue is the trigger for execution |
| **Polling (last resort)** | Provider has no webhook | Worker on interval; still write **idempotent** handlers |

**Best default for reliability:** webhook/cron/user only **creates or updates durable records** and **enqueues**; workers do I/O and call Dify **from the worker** when the model must see fresh data.

### 16.3 Trigger hardening (non-optional)

- **Idempotency:** require `Idempotency-Key` (user/API) or dedupe on `(provider, external_event_id)` stored in PostgreSQL with a **unique constraint**. Assume **at-least-once** delivery everywhere.
- **Correlation:** generate `correlation_id` at entry; pass it to Dify metadata (if supported), logs, and every `tasks` row — speeds up debugging.
- **Timeouts:** webhooks should **ack quickly** and defer heavy work to the queue; never block HTTP on a long LLM chain.
- **Auth:** webhooks use **HMAC / signed JWT / provider verification**; user triggers use **session/JWT**; server-to-Dify uses **API key only on server**.
- **Rate limits:** per-user and per-provider caps at the gateway before enqueue storms.

### 16.4 Workflow patterns that age well

1. **Gate pattern (draft → approve → act)**  
   Trigger A creates **draft** tasks or `pending_send` rows. Trigger B (user approval) flips to executable queue jobs. Use for email, DMs, PayPal.

2. **Plan → fan-out**  
   One Dify run returns N `steps` of type `send_email` with recipient ids; orchestrator creates N **idempotent** jobs (`job_key = hash(command_id, recipient_id, template_version)`).

3. **Router workflow**  
   Dify (or a tiny deterministic router in code) picks **one** of a few orchestrator workflows: `triage`, `outreach`, `paypal_prepare` — each with its own JSON schema version (`schema_version: 3` in payload).

4. **Compensation (lightweight)**  
   If step 3 fails after step 2 succeeded, store **what to undo** in DB; optional worker sends “sorry, retry” — avoid complex distributed sagas until you need them.

5. **Versioning**  
   Name apps/workflows in Dify and store `dify_app_id` + `prompt_version` / `workflow_version` on `commands` so you can replay or debug old runs.

### 16.5 Dify-specific choices

| Need | Prefer |
|------|--------|
| Fixed steps, predictable JSON at the end, less user chit-chat | **Workflow** app |
| Clarifying questions, multi-turn refinement, then “lock plan” | **Chatflow** with a final **“Export plan”** node or tool that emits JSON |
| Same brand voice everywhere | **Shared Knowledge** + small number of reusable **prompt snippets** |
| Tool calls | HTTP tools → **your** NestJS/Express routes that only **enqueue** or **return redacted data** |

### 16.6 Anti-patterns (what breaks in production)

- Webhook handler calls Gmail send or PayPal **directly** with no allowlist or approval record.
- Same Stripe/Gmail event processed twice → double payout or double reply (no idempotency row).
- Dify agent given **service account keys** or **full** OAuth refresh tokens.
- “One giant Dify workflow” for unrelated domains — hard to test and to roll back; prefer **small apps** + orchestrator composition.

### 16.7 Minimal trigger → tables mapping

| Event | Write | Then |
|-------|--------|------|
| User command | `commands` | enqueue `process_command` |
| Cron tick | `commands` with `source=cron` | enqueue domain worker |
| Verified webhook | `ingestion_events` (deduped) + optional `commands` | enqueue normalizer |
| Approval tap | update `tasks` or `transactions` | enqueue **execute_*` job |

This section pairs with **§4 execution flow**, **§7 AI configuration**, and **§15 business map**: triggers start runs; workflows define **order and gates**; Dify fills **language and structure** inside that frame.
