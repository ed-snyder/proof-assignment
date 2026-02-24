# Personalized First-Touch Outreach: Agentic Workflow Design

## Workflow Selected: Personalized First-Touch Outreach

**Why this workflow:** First-touch outreach is where AI creates the most differentiated value. The core challenge — researching a prospect, synthesizing context, and writing a compelling personalized message — is exactly what LLMs excel at. Other BDR workflows (routing, scheduling, list building) are fundamentally conditional logic and data plumbing. This is the one where intelligence actually matters, and where automation has the most direct impact on pipeline creation.

---

## Architecture Overview

**Orchestration:** n8n (workflow engine + scheduling + error handling + inter-tool connectivity)
**Enrichment:** Clay (firmographic, technographic, personal context)
**AI Layer:** Anthropic Claude (research synthesis, email generation, quality gate)
**Execution:** Outreach (email delivery + sequence tracking)
**CRM:** Salesforce (account/contact records, activity logging)
**Intent:** 6Sense (buying stage, intent topics, account surge signals)
**Product Data:** Internal data warehouse (Redshift) — signups, feature activation, usage patterns, drop-offs
**Config:** `prompt.md` — a living prompt document the GTME maintains that governs agent behavior across all AI steps

### Design Principle: Agent-Monitored, Not Human-in-the-Loop

The system operates autonomously by default. A supervisory agent monitors every step of the workflow and only escalates to the GTM Engineer when it detects an anomaly. The GTME is not reviewing every email — they're managing the system, not doing the work. Think of it like a self-driving car: the human doesn't steer, but they get an alert if the car encounters something it can't handle.

---

## Step-by-Step Workflow

### Step 0: Input — Dual-Source Contact Ingestion
**What triggers it:** Two parallel n8n trigger nodes fire on independent schedules

#### Source A: 6Sense Intent Surges
- **n8n node:** Schedule Trigger (daily, 6am) → HTTP Request node hitting the 6Sense API (`/v3/accounts/intent`)
- **What it pulls:** Accounts showing intent surges on Proof-relevant buying topics (e.g., "digital notarization," "remote online closing," "identity verification," "eSignature alternatives")
- **Filters applied in n8n (IF node):** Buying stage = "Decision" or "Purchase" (not "Awareness"); intent score > threshold; account is in Proof's target segment (Salesforce lookup via SF node to confirm account exists + not already in active sequence)
- **Contact resolution:** 6Sense returns account-level data, not contacts. n8n passes surging accounts to Clay (via HTTP Request to Clay's API) to find the right contacts — typically VP/Director-level in ops, legal, compliance, or product depending on industry.

#### Source B: Product / PLG Signals
- **n8n node:** Schedule Trigger (daily, 6am) → Postgres/Redshift node querying the internal data warehouse
- **Query logic:** Pulls users who hit meaningful product milestones in the last 24 hours:
  - New pro signups (trial started)
  - Feature activation thresholds (e.g., completed first notarization, uploaded documents, invited team members)
  - Usage spikes (transaction volume increase week-over-week)
  - Drop-off signals (signed up but no activity in 7 days, or previously active and usage declined 50%+)
- **Filters applied in n8n (IF node):** Exclude existing customers on paid plans (Salesforce lookup via SF node — check Opportunity stage != "Closed Won"); exclude contacts already in an active Outreach sequence (Outreach API lookup)

#### Merge & Deduplicate
- **n8n node:** Merge node (append mode) combines both source outputs → a Remove Duplicates node deduplicates on email address
- **n8n node:** A Code node tags each contact with their `signal_source` (intent, product_signup, feature_activation, usage_spike, drop_off) and `signal_strength` (derived from 6Sense score or product activity level). This metadata flows through the entire workflow and influences messaging later.

**Output:** A deduplicated batch of contacts with signal source + strength, ready for enrichment

**What a BDR used to do:** BDRs received these leads — surfaced via dashboards, 6Sense alerts, or Salesforce reports. The leads were handed to them. But what happened next was entirely manual: the BDR had to decide who to prioritize, research each contact, figure out what to say, and write the email. That's what this system replaces.

---

### Step 1: Enrichment (Clay via n8n)
**n8n connection:** HTTP Request node → Clay API (`/v1/people/enrich` and `/v1/companies/enrich`)

n8n sends each contact to Clay with their email and company domain. Clay runs its enrichment waterfall across multiple data providers and returns structured data.

**Clay pulls:**
- Firmographic: company size, industry, funding stage, revenue, HQ location
- Technographic: current stack (especially notarization, identity verification, document workflow, eSignature tools — critical for competitive positioning)
- Personal context: role, seniority, tenure, LinkedIn headline, recent LinkedIn posts/activity
- Company context: recent news, job postings (especially ops/legal/compliance hires), funding rounds, M&A activity

**n8n processing after Clay returns:**
- A Code node merges Clay's enrichment data with the signal data from Step 0 into a single unified contact object
- An IF node checks enrichment completeness: if critical fields are missing (no title, no company industry), the contact is routed to a "low enrichment" holding queue for the GTME to review rather than generating a poorly-informed email

**Output:** Enriched contact record with signal context + firmographic/technographic/personal data, stored on the n8n workflow item

**What a BDR used to do:** Spent 10-15 minutes per prospect on LinkedIn, the company website, news articles. Got maybe 3-4 data points. Clay does this in seconds across 20+ sources.

---

### Step 2: Research Synthesis (Anthropic Claude via n8n)
**n8n connection:** HTTP Request node → Anthropic Messages API (`/v1/messages`). The system prompt is loaded from `prompt.md` (stored as a static file node or pulled from a shared drive). The enriched contact JSON is injected as the user message.

This step has two jobs:

#### 2a: Metadata Transformation
The agent generates clean, conversational metadata for use in outreach. Raw CRM/enrichment data is not email-ready.

| Raw Field | Transformed Output | Example |
|---|---|---|
| Title | Personalized role descriptor | "Senior Software Engineer Lvl 3" → "lead SWE" |
| Company (legal name) | Conversational company name | "Acme Corporation, Inc." → "Acme" |
| Department | Functional context | "Revenue Operations & Strategy" → "RevOps" |
| Industry | Plain-language vertical | "Computer Software - SaaS B2B" → "B2B SaaS" |

**Why this matters:** If an email says "Hello, Senior Software Engineer Level 3 at Acme Corporation, Inc." it reads like a mail merge. Transforming metadata into how a human would actually reference someone's role/company is what makes personalization feel real.

#### 2b: Hypothesis Generation
The agent generates a structured brief answering three questions. Critically, the `signal_source` from Step 0 shapes the angle:

1. **Why Proof?** — Based on the prospect's industry, tech stack, and company context, what specific Proof capability is most relevant? (e.g., "They're in real estate closings and still using wet signatures — Proof's remote online notarization eliminates their biggest bottleneck")

2. **Why this person?** — Based on their role and department, why are they the right contact? What do they care about? (e.g., "As head of ops, they own the closing workflow and feel the pain of scheduling in-person notarizations")

3. **Why now?** — This is driven by the signal source:
   - *Intent surge:* "6Sense shows active research on 'digital closing solutions' — they're evaluating options right now"
   - *Product signup:* "They just signed up for a Proof trial 2 days ago — momentum is high, they're in evaluation mode"
   - *Feature activation:* "They completed their first notarization on the platform yesterday — this is the moment to expand the conversation from trial to enterprise"
   - *Usage drop-off:* "They signed up 10 days ago but haven't completed onboarding — they need a nudge before they churn from the trial"
   - *Usage spike:* "Transaction volume tripled this week — they're hitting scale and may need enterprise features"

**Output:** Claude returns a structured JSON object with transformed metadata + hypothesis brief. n8n parses this via a Code node and attaches it to the workflow item.

**Governed by:** `prompt.md` — contains the system prompt, transformation rules, hypothesis framework, tone guidelines, and example outputs. This is the single source of truth the GTME updates when they want to change agent behavior.

**What a BDR used to do:** The best BDRs did this intuitively for their top 5 accounts. Most didn't do it at all. This system does it for every single contact, consistently.

---

### Step 3: Email Generation (Anthropic Claude via n8n)
**n8n connection:** HTTP Request node → Anthropic Messages API. Separate call from Step 2 (different system prompt optimized for writing, not analysis).

**How it works:**
- The GTME maintains a library of **outreach templates** stored in n8n as JSON (via a Code node or external file). Each template defines the *structure* of the email: what goes in the opening line, the body, and the CTA. Templates are mapped to `signal_source` — an intent-surge contact gets a different template than a trial-drop-off contact.
- n8n's Code node selects the appropriate template based on the contact's `signal_source` and persona
- Claude receives: (1) the selected template, (2) the full research synthesis from Step 2, (3) the messaging framework from `prompt.md`
- Claude generates the email, filling in the template structure with personalized content from the research

**Constraints (enforced in prompt + validated in Step 4):**
- Total email body < 250 words
- No generic filler ("I hope this email finds you well")
- Must reference at least one specific detail from the hypothesis (why now / why them)
- CTA is low-friction (not "book a 30 min demo" — more like "worth a look?" or "open to seeing how X works?")
- Tone matches `prompt.md` guidelines

**Output:** Draft email (subject line + body) attached to the contact record in the n8n workflow

**What a BDR used to do:** Wrote emails from scratch or lightly edited templates. Quality varied wildly by rep. Top performers wrote great emails but could only do 20-30/day. This system produces consistently good emails at 100x the volume.

---

### Step 4: Quality Gate (Supervisory Agent — Anthropic Claude via n8n)
**n8n connection:** HTTP Request node → Anthropic Messages API. Third Claude call, with a *different* system prompt optimized for evaluation, not generation. This is a separate model call acting as an independent reviewer.

**This is the core of the agentic design.** The supervisory agent reviews each email against a checklist:

**Automated checks:**
- [ ] Word count < 250
- [ ] No hallucinated company details (cross-referenced against the enrichment data passed in as context)
- [ ] Personalization references a real, specific data point (not vague hand-waving)
- [ ] Tone matches guidelines in `prompt.md`
- [ ] CTA is present and low-friction
- [ ] No spam trigger words
- [ ] Subject line is specific, not generic
- [ ] Transformed metadata reads naturally (no raw CRM artifacts like "Acme Corporation, Inc.")
- [ ] Signal source is reflected in the messaging (an intent-surge email shouldn't read like a product-signup email)

**Claude returns a structured JSON verdict:** `{ "result": "PASS" | "SOFT_FAIL" | "HARD_FAIL", "issues": [...], "suggestions": [...] }`

**n8n routing (Switch node):**

| Result | n8n Path | Action |
|---|---|---|
| PASS | → Outreach send node | Email moves to Step 5 automatically |
| SOFT FAIL | → Loop back to Step 3 | Agent auto-rewrites with the issue flagged in the prompt. n8n tracks retry count — max 2 retries before escalating to HARD FAIL |
| HARD FAIL | → GTME review queue | Email is flagged and routed to a dedicated n8n webhook/dashboard or Slack alert with the email, the enrichment data, and a summary of what failed and why |

**GTME feedback loop:** When the GTME reviews a hard-fail, they do two things:
1. Fix the individual email (or discard it)
2. Update `prompt.md` to address the *systemic* issue so the same failure doesn't recur. For example, if the agent keeps hallucinating product features, the GTME adds a "Do not reference features not in this list: [...]" constraint to `prompt.md`.

**What a BDR used to do:** A sales manager might review 5% of outbound emails in a weekly 1:1. The supervisory agent reviews 100% in real time.

---

### Step 5: Execution (Outreach via n8n)
**n8n connection:** HTTP Request node → Outreach REST API (`/v2/prospects` to upsert the contact, `/v2/sequences/{id}/sequenceStates` to add them to a sequence, or `/v2/mailings` for one-off sends)

**What happens:**
- n8n upserts the contact in Outreach with all enrichment data as custom fields
- Contact is added to the appropriate Outreach sequence (mapped by `signal_source` — intent leads go into one sequence, PLG leads into another)
- The generated first-touch email is set as the personalized first step; subsequent steps use Outreach's native sequence logic
- Activity is logged back to Salesforce via n8n's Salesforce node (creates a Task on the contact record with email body + signal context)
- n8n updates its internal tracking: email sent, timestamp, sequence ID, contact ID

**Output:** Email delivered. Outreach begins tracking opens, clicks, replies.

---

### Step 6: Response Handling & Monitoring
**n8n connection:** Outreach fires a webhook on reply events → n8n Webhook Trigger node catches it

**n8n routing (Switch node based on reply sentiment — optionally scored by a quick Claude call):**

**Positive reply:** n8n creates a Salesforce Task on the contact + sends a Slack message (via Slack node) to the assigned AE with full context: the research brief from Step 2, the email sent, the reply, and the original signal source. The AE walks into the conversation knowing *why* this person was contacted and *what* they care about.

**Negative reply / objection:** n8n logs the objection text to a Salesforce custom field and tags it by category. Over time, this builds an objection corpus that feeds back into the system for messaging refinement (connects to Assessment Section 2).

**No reply:** Contact remains in the Outreach sequence for automated follow-up touches (connects to multi-step follow-up — a separate workflow).

---

## n8n Workflow Structure (Node-by-Node)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  n8n Workflow: Personalized First-Touch Outreach                        │
│                                                                         │
│  TRIGGERS (parallel)                                                    │
│  ┌──────────────────┐     ┌───────────────────────┐                     │
│  │ Schedule Trigger  │     │ Schedule Trigger       │                    │
│  │ (Daily 6am)       │     │ (Daily 6am)            │                    │
│  │                   │     │                        │                    │
│  │ → HTTP Request    │     │ → Redshift Node        │                    │
│  │   6Sense API      │     │   Product data query   │                    │
│  │   /accounts/intent│     │   (signups, activation, │                   │
│  │                   │     │    dropoff, spikes)     │                    │
│  │ → IF Node         │     │                        │                    │
│  │   Filter: stage,  │     │ → IF Node              │                    │
│  │   score, not in   │     │   Filter: not existing │                    │
│  │   active sequence │     │   customer, not in     │                    │
│  │                   │     │   active sequence      │                    │
│  │ → Clay API        │     │                        │                    │
│  │   Resolve contacts│     │                        │                    │
│  └────────┬─────────┘     └──────────┬────────────┘                     │
│           │                          │                                   │
│           └──────────┬───────────────┘                                   │
│                      ▼                                                   │
│             ┌────────────────┐                                           │
│             │ Merge Node     │                                           │
│             │ + Deduplicate  │                                           │
│             │ + Code Node:   │                                           │
│             │   tag signal   │                                           │
│             │   source +     │                                           │
│             │   strength     │                                           │
│             └───────┬────────┘                                           │
│                     ▼                                                    │
│  ENRICHMENT                                                              │
│  ┌──────────────────────────┐                                           │
│  │ HTTP Request → Clay API  │                                           │
│  │ /people/enrich           │                                           │
│  │ /companies/enrich        │                                           │
│  │                          │                                           │
│  │ → Code Node: merge       │                                           │
│  │   enrichment + signal    │                                           │
│  │                          │                                           │
│  │ → IF Node: enrichment    │                                           │
│  │   completeness check     │──── LOW ──→ [GTME holding queue]          │
│  └──────────┬───────────────┘                                           │
│             │ COMPLETE                                                   │
│             ▼                                                            │
│  RESEARCH SYNTHESIS                                                      │
│  ┌──────────────────────────┐                                           │
│  │ HTTP Request → Claude API│                                           │
│  │ System prompt: prompt.md │                                           │
│  │ User msg: enriched JSON  │                                           │
│  │                          │                                           │
│  │ Output:                  │                                           │
│  │  - Transformed metadata  │                                           │
│  │  - Hypothesis brief      │                                           │
│  │                          │                                           │
│  │ → Code Node: parse JSON  │                                           │
│  └──────────┬───────────────┘                                           │
│             ▼                                                            │
│  EMAIL GENERATION                                                        │
│  ┌──────────────────────────┐                                           │
│  │ Code Node: select        │                                           │
│  │ template by signal_source│                                           │
│  │                          │                                           │
│  │ → HTTP Request → Claude  │                                           │
│  │   System prompt: prompt.md                                           │
│  │   + template + research  │                                           │
│  │                          │                                           │
│  │ Output: email subject    │                                           │
│  │ + body (< 250 words)     │                                           │
│  └──────────┬───────────────┘                                           │
│             ▼                                                            │
│  QUALITY GATE                                                            │
│  ┌──────────────────────────┐                                           │
│  │ HTTP Request → Claude    │                                           │
│  │ (evaluator prompt)       │                                           │
│  │                          │                                           │
│  │ Input: email + enrichment│                                           │
│  │ + research synthesis     │                                           │
│  │                          │                                           │
│  │ Output: PASS/SOFT/HARD   │                                           │
│  │                          │                                           │
│  │ → Switch Node:           │                                           │
│  │   PASS ──────────────────┼──→ [Outreach Send]                        │
│  │   SOFT_FAIL ─────────────┼──→ [Loop back to Email Gen, max 2x]       │
│  │   HARD_FAIL ─────────────┼──→ [Slack alert + GTME review queue]      │
│  └──────────────────────────┘                                           │
│                                                                         │
│  EXECUTION                                                               │
│  ┌──────────────────────────┐                                           │
│  │ HTTP Request → Outreach  │                                           │
│  │ Upsert prospect          │                                           │
│  │ Add to sequence          │                                           │
│  │                          │                                           │
│  │ → Salesforce Node        │                                           │
│  │   Log activity as Task   │                                           │
│  └──────────────────────────┘                                           │
│                                                                         │
│  RESPONSE HANDLING                                                       │
│  ┌──────────────────────────┐                                           │
│  │ Webhook Trigger ←        │                                           │
│  │ Outreach reply event     │                                           │
│  │                          │                                           │
│  │ → Switch Node:           │                                           │
│  │   POSITIVE → SF Task     │                                           │
│  │              + Slack AE  │                                           │
│  │   NEGATIVE → SF field    │                                           │
│  │              + objection  │                                           │
│  │              log         │                                           │
│  │   NO REPLY → stays in    │                                           │
│  │              sequence    │                                           │
│  └──────────────────────────┘                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Measuring Performance

### Outreach Performance Metrics (Is it generating pipeline?)

| Metric | Target | Frequency | Signal to Iterate |
|---|---|---|---|
| Open rate | > 45% | Daily | Below 35% → subject line or deliverability issue |
| Reply rate | > 8% | Daily | Below 5% → messaging or targeting problem |
| Positive reply rate | > 50% of replies | Weekly | Low → CTA or value prop isn't landing |
| Meeting booked rate | > 3% of sends | Weekly | Low → qualification or handoff problem |
| Pipeline created ($) | Tracked per cohort | Monthly | Flat/declining → systemic workflow issue |
| AE acceptance rate | > 80% of meetings | Weekly | Low → qualification logic needs tightening |

### Workflow & Agent Health Metrics (Is the system working?)

These are just as important. A broken workflow produces zero pipeline silently.

| Metric | What It Tells You | Frequency |
|---|---|---|
| Enrichment fill rate | % of fields Clay successfully populates per contact | Daily |
| Synthesis completion rate | % of contacts that get a full research brief without error | Daily |
| Quality gate pass rate | % of emails that pass on first attempt | Daily |
| Soft fail → pass rate | % of auto-rewrites that succeed | Daily |
| Hard fail rate | % of emails requiring GTME intervention | Daily |
| Hard fail categories | What types of errors are recurring (hallucination, tone, length, etc.) | Weekly |
| End-to-end throughput | Contacts in → emails sent per day | Daily |
| Latency | Time from trigger to email sent | Daily |
| `prompt.md` update frequency | How often the GTME is tuning the system | Weekly |
| Error rate by step | Which n8n node is failing and how often | Daily |

### Metrics by Signal Source

Because contacts enter from different signals, performance should be segmented:

| Signal Source | Key Comparison |
|---|---|
| 6Sense intent surge | Do high-intent accounts reply at higher rates? If not, our intent topic selection may be off |
| Product signup (new) | Are we reaching them fast enough? Speed-to-first-touch matters most here |
| Feature activation | Does referencing their specific product activity improve reply rates vs generic messaging? |
| Usage drop-off | Is re-engagement messaging pulling them back, or are these dead leads? Helps decide whether to keep this signal active |
| Usage spike | Are these converting to enterprise conversations? Validates the PLG → sales-assist motion |

### Key Iteration Signals

- **Hard fail rate climbing:** The agent is struggling with a new segment or data pattern → GTME investigates and updates `prompt.md`
- **High open rate but low reply rate:** Subject lines work but email body isn't compelling → revisit hypothesis generation or template structure
- **Low enrichment fill rate:** Clay source is degraded or a new segment doesn't have good coverage → add fallback enrichment sources or secondary providers
- **AE acceptance rate dropping:** Qualification criteria are too loose → tighten scoring logic in Step 0 or add a pre-send qualification check
- **Throughput drop:** An API is rate-limited or an n8n node is failing → check n8n execution logs
- **One signal source consistently underperforming:** Deprioritize or rethink the messaging approach for that signal. Not all signals are equal — the data should tell us which ones to double down on.

---

## Where AI Adds the Most Value

1. **Research synthesis** — This is the step no BDR could do at scale. Turning 20 raw data points into a coherent hypothesis about why to reach out *right now* is genuine intelligence work.
2. **Email generation** — Not just "personalization" (anyone can do `{{first_name}}`). Actual contextual messaging that connects a prospect's situation to Proof's value.
3. **Quality gate** — 100% review rate with consistent criteria. No BDR manager could review every email. And because the supervisory agent catches problems at the individual email level, the GTME can focus on fixing the *system* rather than policing output.

## Where Humans Stay in the Loop

The GTME is not in the workflow. The GTME is *above* it.

1. **GTME manages the system** — Updates `prompt.md`, reviews hard fails, monitors agent health metrics, iterates on templates. The GTME's job is to make the system better, not to do the system's job.
2. **AEs own the relationship** — Once a meeting is booked, the human takes over with full context provided by the system. The AE never cold-walks into a conversation.
3. **Revenue Marketing owns messaging strategy** — Templates and `prompt.md` tone guidelines should be co-developed with marketing (connects to Assessment Section 4).
