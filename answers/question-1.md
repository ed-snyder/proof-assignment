# Question 1: Agentic Workflow Design

## The Workflow and Why We Picked It

We're replacing personalized first-touch outreach. The full cycle: a new signal comes in, the prospect gets researched, a tailored email gets written, and it lands in their inbox. We picked this one because it's the only BDR workflow where intelligence actually matters. Other candidates like lead routing, list building, or scheduling are just conditional logic. You can automate those with Zapier. First-touch outreach requires taking raw data about a person and their company, forming a point of view about why they should care about your product right now, and writing something that feels like a human did the research. That's a reasoning task, not a rules task, and it's what LLMs are built for.

Why it's agentic: the system makes judgment calls and corrects itself. It doesn't just move data through a pipeline. It synthesizes research into a hypothesis, writes an email grounded in that hypothesis, then has a separate AI evaluate the email against quality standards. If it fails, the system rewrites it. If it still can't get it right, it escalates to a human. That self-correction loop (generate, evaluate, retry, escalate) is what separates this from standard automation. No human reviews every email. The system monitors itself.

## Step-by-Step Walkthrough

**Trigger.** Two signal sources fire in parallel every morning at 6am via n8n. Source one is 6Sense intent data: accounts actively researching Proof-relevant topics like digital notarization or remote closing. The system filters to only Decision/Purchase stage accounts above a score threshold, and not already in an active sequence. Since 6Sense returns accounts (not people), it passes those to Clay to resolve the right contacts. Source two is product usage data from Proof's warehouse: trial signups, feature activations, usage spikes, and drop-offs from the last 24 hours. Same filters apply. Both streams merge, deduplicate on email, and each contact gets tagged with their signal source and strength. That tag shapes everything downstream. A BDR used to receive these leads through dashboards, but what happened next (who to prioritize, what to say, how to say it) was all manual.

**Enrichment.** Each contact goes to Clay, which runs its data waterfall across providers and returns firmographics, technographics (especially their current notarization and doc workflow tools), personal context from LinkedIn, and company context like recent news and job postings. The system merges this with the signal data. If critical fields are missing, that contact goes to a holding queue instead of getting a bad email. A BDR used to spend 10-15 minutes per prospect on LinkedIn and company websites, getting maybe 3-4 data points. Clay does it in seconds across 20+ sources.

**Research Synthesis.** The enriched contact JSON goes to Claude with a system prompt from prompt.md, the living document the GTM Engineer maintains. Claude transforms raw CRM metadata into natural language ("Senior Software Engineer Level 3 at Acme Corporation, Inc." becomes "lead SWE at Acme") and generates a hypothesis brief: Why Proof (what capability fits their stack and industry), Why this person (what they care about given their role), and Why now (driven directly by the signal source). The best BDRs did this intuitively for their top 5 accounts. Most didn't do it at all. This system does it for every contact, consistently.

**Email Generation.** A separate Claude call takes the research synthesis plus a template selected by signal source and writes the email. Constraints are enforced: under 250 words, no filler, must reference a specific detail from the hypothesis, low-friction CTA. BDRs wrote emails from scratch or lightly edited templates. Quality varied wildly. Top performers could do 20-30 good ones a day.

**Quality Gate.** A third Claude call (different prompt, optimized for evaluation) reviews each email: word count, hallucination checks against enrichment data, personalization specificity, tone, spam triggers. It returns PASS, SOFT_FAIL, or HARD_FAIL. Pass goes straight to send. Soft fail loops back for a rewrite (max 2 retries). Hard fail goes to the GTM Engineer with a Slack alert and full context. When the GTME reviews a hard fail, they fix the email and update prompt.md so the systemic issue doesn't recur. A sales manager used to spot-check maybe 5% of emails in a weekly 1:1. This reviews 100% in real time.

**Execution.** Passing emails get pushed to Outreach, contacts are added to the right sequence by signal source, and activity logs back to Salesforce. When replies come in via webhook, positive ones trigger a Slack message to the AE with the full research brief, the email sent, and the reply. The AE walks in knowing exactly why this person was contacted. Negative replies get logged to build an objection corpus. No replies stay in the sequence.

## Where AI Beats Typical Automation (and Where Humans Stay)

AI adds the most value in three places. Research synthesis is the biggest: turning 20 data points into a hypothesis about why to reach out is reasoning work that no IF/THEN logic can replicate. Email generation is the second: this is contextual messaging, not mail merge personalization. The quality gate is the third: 100% review with consistent criteria and self-correction at scale.

Humans stay involved in three ways. The GTM Engineer manages the system (updates prompt.md, reviews hard fails, monitors health metrics). AEs own the relationship from the moment a meeting books, with full context handed to them. Revenue Marketing co-owns the messaging strategy so outbound stays aligned with campaigns.

## Metrics and Iteration Signals

On the outreach side, we track open rate (target 45%+), reply rate (target 8%+), positive reply rate (50%+ of replies), meeting booked rate (3%+ of sends), and AE acceptance rate (80%+). Each one points to a specific fix if it drops: low opens mean subject line or deliverability problems, low replies mean messaging or targeting is off, low AE acceptance means qualification is too loose.

On the system health side, we track enrichment fill rate, quality gate pass rate, hard fail rate and categories, throughput, and latency. A broken workflow produces zero pipeline silently, so these matter just as much.

Everything gets segmented by signal source. Do intent accounts reply more? Are we reaching trial signups fast enough? Are drop-off emails actually re-engaging people? This tells us which signals to invest in and which to cut. If hard fail rates climb, the agent is struggling with a new pattern. If one signal source consistently underperforms, the data is telling us to rethink it or stop.
