# Question 2: Signal-Based Prioritization & Enrichment Logic

## The Two Signals and How They Flow

We selected 6Sense intent surges and product usage data from Proof's own data warehouse. Together they cover the full intent spectrum: 6Sense catches accounts that are thinking about a solution, and product data catches people who are already using one. Everything else in the stack (HubSpot engagement, Salesforce stage changes) is useful context, but these two are the primary activation triggers because they represent the clearest buying signals.

### Signal 1: 6Sense Intent Surges

6Sense is account-level, not contact-level. When it says "Acme Corp is surging on digital notarization," it doesn't tell you who at Acme to email. So the integration has to solve that gap.

The flow starts with an n8n schedule trigger at 6am daily. It hits the 6Sense API to pull accounts showing intent surges on Proof-relevant topics (digital notarization, remote online closing, identity verification, eSignature alternatives). The recommended approach is to pre-build a segment in the 6Sense UI that filters for buying stage = Decision or Purchase, intent score above 60, and profile fit = Strong or Moderate. Then pull segment membership via the API rather than scoring 50K accounts individually.

For each surging account, we need actual contacts. The 6Sense People Search API handles this. We POST to `/v2/search/people` with the account domain and filters for Director/VP/C-level in Operations, Legal, Compliance, or Product. It returns email, title, seniority, LinkedIn URL. Rate limit is 10 queries/sec, so a daily batch of ~100 surging accounts finishes in about 10 seconds.

Those contacts go to Clay for deeper enrichment (LinkedIn activity, company news, tech stack, hiring signals), then get written to Salesforce. The field mapping is straightforward: 6Sense fields like `company_buying_stage`, `company_intent_score`, and `company_profile_fit` map to custom account fields in Salesforce. Contact-level fields from Clay (tech stack, company news, signal source tag) map to custom contact fields. Everything upserts on domain for accounts and email for contacts.

Finally, contacts get pushed to Outreach via the REST API. Outreach doesn't have a native upsert, so you lookup by email first, then create or update. The prospect gets added to an intent-surge-specific sequence by creating a `sequenceState` resource linking the prospect, the sequence, and a mailbox.

### Signal 2: Product Usage Data

This one's simpler on the integration side because it's first-party data. An n8n schedule trigger at 6am queries Proof's Redshift warehouse (Postgres-compatible, so n8n's Postgres node works natively). The query pulls four types of events from the last 24 hours: new pro trial signups, feature activations (first notarization completed), usage spikes (transaction volume 2x+ week-over-week), and drop-offs (signed up 7+ days ago with no recent activity).

Each contact type gets a base signal strength score: usage spikes at 80, feature activations at 75, new signups at 60, drop-offs at 50. Bonuses for things like company size or target industry. These scores set priority order within the daily batch.

Before activation, every contact gets checked against Salesforce: not an existing customer, not already in an active sequence, not on a do-not-contact list. Then through the same Clay enrichment, Salesforce write, and Outreach activation path as Signal 1, but into different sequences mapped to the signal type.

### Messaging Approach: Hybrid

The messaging is hybrid, not static or fully dynamic. The static layer is Outreach templates that define the email structure. The GTME creates and controls these. The dynamic layer is AI-generated content from the personalized outreach workflow (Question 1) that gets written into Outreach custom fields. The template references those fields using `{{prospect.custom4}}`, `{{prospect.custom5}}`, etc. So the structure is fixed and the content is personalized per contact.

This matters because product signals demand completely different messaging than intent signals. A trial signup gets a helpful, non-salesy welcome that references their use case. A feature activation gets a "here's what teams like yours do next" expansion message. A usage spike gets an enterprise-scale conversation. A drop-off gets friction removal and a walkthrough offer. These aren't just different first emails; they're distinct Outreach sequences with different follow-up cadences and escalation logic.

## Mining Closed-Lost Data and Gong Recordings

This is the feedback loop that makes everything smarter over time. It runs monthly as a batch analysis job, not a real-time workflow.

The technical challenge is that Gong has no way to filter calls by deal outcome. You can't say "give me calls from closed-lost deals." So the pipeline works in stages: first, query Salesforce via SOQL for all closed-lost opportunities from the last 12 months (returns opp IDs, account info, industry, size, loss reason, competitor). Second, pull Gong calls with CRM context using `POST /v2/calls/extensive`, which returns calls with linked opportunity IDs. Third, match calls to closed-lost opp IDs client-side. Fourth, pull transcripts for matched calls via `POST /v2/calls/transcript`. Fifth, feed transcripts plus deal metadata to Claude in batches of 5 for structured analysis.

This is better handled as a Python script than an n8n workflow because it involves cross-system joins, pagination, rate limiting, and LLM batch processing.

Claude extracts five things from each batch: primary objections (categorized by type like pricing, feature gap, competitor preference, timing), pain points mentioned, competitor references and how they were perceived versus Proof, segment signals (does this industry/size look like one we should deprioritize), and messaging gaps (did our sales conversation miss something the outbound email could have addressed earlier).

After the batch analyses, a synthesis call combines everything into a single intelligence brief: top objection patterns ranked by frequency, segments to deprioritize, segments to double down on, specific messaging changes, and competitor positioning.

### What We'd Learn About Deprioritization

The brief would surface patterns like: companies under 50 employees in a certain vertical consistently churn because they don't have the transaction volume to justify Proof. Or a specific industry keeps raising a regulatory objection that Proof genuinely can't solve. Those segments get added to the n8n suppression filters so we stop wasting enrichment credits and outreach on them.

On the messaging side, if prospects keep raising the same objection (say, "we need on-prem deployment"), that gets added to prompt.md so the research synthesis agent knows to check for it. If the prospect's profile matches the objection pattern, the system either preemptively addresses it in the email or flags the contact for suppression if it's a true dealbreaker.

### The Feedback Loop Technically

The script outputs `closed_lost_intelligence.json`. The GTME reviews it and takes three actions: update n8n suppression rules (add IF node filters for bad segments), update prompt.md with objection preemption rules and refined hypothesis frameworks, and update Outreach templates to address messaging gaps. The result is that Motions 1 and 2 get smarter every month with better targeting, better messaging, and fewer wasted emails.

### Refresh Cadence

The full Gong analysis runs monthly. With ~150 closed-lost opps over 12 months, that's roughly 12 new data points per month, which is enough to spot trends without overreacting. The GTME reviews the brief the day after the run and implements changes that week. There's also a lightweight weekly SOQL check for any high-value losses (say, $100K+ deals lost to a specific competitor) that warrant immediate attention rather than waiting for the monthly cycle.

## Defining and Measuring Success

The core metrics, in order of importance:

Signal-to-activation rate (what % of surfaced signals actually result in an email being sent) targets above 70%, reviewed weekly. Signal-to-reply rate targets above 8% overall, segmented by signal source, reviewed weekly. Signal-to-meeting rate targets above 3%, reviewed weekly. Signal-to-pipeline rate (actual dollars) targets above 1%, reviewed monthly. AE acceptance rate targets above 80%, reviewed weekly. Enrichment coverage targets above 85%, reviewed daily. Time-to-activation (signal fires to email sent) targets under 24 hours, reviewed daily.

The most important view is the signal source comparison table: reply rate, meeting rate, pipeline dollars, and AE acceptance broken out by 6Sense intent surge, product signup, feature activation, usage spike, and drop-off. This tells you which signals are actually worth acting on. If drop-off contacts consistently produce 1% reply rates while usage spikes produce 15%, you reallocate accordingly.

Changes happen at three cadences. Daily is automated monitoring: workflow execution rates, enrichment fill, signal volume, quality gate pass/fail. Weekly is the GTME reviewing reply and meeting rates by signal source, AE feedback, and making tactical adjustments to scoring thresholds, template copy, and filters. Monthly is the strategic review: run the Gong analysis, review the intelligence brief with AEs and Revenue Marketing, update suppression lists and prompt.md, and decide whether to add or remove signal sources based on performance data.
