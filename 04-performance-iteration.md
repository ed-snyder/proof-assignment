# Performance Iteration & Cross-Functional Collaboration

## The Situation (Week 3)

The outbound system is live. There is no BDR team to fall back on. Here's what we're seeing:

| Signal | Status |
|---|---|
| Open rates on personalized emails | 45% (strong) |
| Reply rates | 3% (low) |
| AE meeting quality feedback | "Low quality" — prospects aren't well-qualified or don't know why they're on the call |
| Revenue Marketing | Says outbound messaging doesn't align with campaign positioning |
| AE trust | One AE has started manual prospecting on the side |

---

## Part 1: Diagnosing Root Causes

### The Key Insight: These Problems Are Connected

45% open rate means subject lines work and deliverability is healthy. That's not the problem. The problem is downstream — the email body isn't earning a reply, and when it does earn a meeting, the meeting quality is poor. These aren't four separate issues. They're symptoms of the same root causes.

### Investigation Priority Order

#### Investigation 1: Messaging Misalignment (Most Likely Primary Cause)

Revenue Marketing already told us the outbound copy doesn't match their campaign positioning. This isn't a separate issue from the low reply rate — it's probably *the* cause.

If Proof's marketing is positioning around one value prop (say, "eliminate notarization fraud risk") and the outbound emails are leading with a different one (say, "speed up your closing workflow"), prospects who've seen Proof's brand get a dissonant first touch. It doesn't resonate because it doesn't match the narrative they've already been exposed to.

**Data to pull:**
- Revenue Marketing's campaign positioning document for the target segment — the actual copy, headlines, and value props they're using
- 20-30 sample outbound emails from the system — the actual emails sent, not the templates
- Side-by-side comparison: where specifically do the messaging angles, language, and value props diverge?

**What I'd look for:**
- Are we leading with a different value prop than marketing?
- Are we using different language for the same concepts? (e.g., marketing says "digital identity verification" but outbound says "remote notarization")
- Is the tone different? (Marketing is polished/brand-voice, outbound is casual/salesy?)

#### Investigation 2: Personalization Depth

High open rate + low reply rate often means the subject line got attention but the email body didn't earn a response. The personalization might be surface-level.

**Data to pull:**
- Read 20-30 actual sent emails. Not the templates — the rendered output.
- Check the quality gate logs: what's the pass rate? What's the hard fail rate? If pass rate is 95%+, the quality gate might be too lenient.

**What I'd look for:**
- Is the personalization actually specific? "I noticed you just posted 3 ops roles — sounds like you're scaling your closing workflow" is good. "As a leader in real estate, you know closings are complex" is generic filler wearing a personalization costume.
- Is the "Why now?" hypothesis landing? Or is it vague?
- Is the metadata transformation working? Are emails saying "Senior Software Engineer Level 3 at Acme Corporation, Inc." or "lead SWE at Acme"?

**If this is the problem:** Update `prompt.md` with stricter personalization requirements and better examples. Add a quality gate check specifically for personalization specificity. Review whether Clay enrichment is returning enough usable data.

#### Investigation 3: CTA and Expectation Setting

If the CTA is vague ("would you be open to a quick chat?"), two things happen:
1. Most people don't reply because there's no specific reason to
2. The few who do say yes show up not knowing why they're on the call — which is exactly what AEs are reporting

**Data to pull:**
- Categorize all replies from the past 3 weeks: positive/negative/neutral. For positive replies that led to meetings, what did the prospect actually say?
- For the meetings AEs flagged as "low quality," pull the original outbound email. What was the CTA? What expectation did it set?

**What I'd look for:**
- Is the CTA specific enough? "Would it be useful to see how [Company X] cut their closing time by 40%?" sets a clear expectation. "Open to connecting?" does not.
- Does the email body tell the prospect what the meeting will be about? If not, the prospect walks in cold and the AE has to start from scratch.

**If this is the problem:** Update templates to include a specific, value-oriented CTA that sets a clear meeting expectation. Update `prompt.md` to require the CTA to reference the hypothesis.

#### Investigation 4: AE Context Gap

The "low quality" meeting feedback might partly be an information problem on the AE side, not a lead quality problem.

**Data to pull:**
- What does the AE actually see before a meeting? Is the research brief (from Assessment 1, Step 2) making it to them via the Slack notification / Salesforce task?
- Pull Gong recordings of the first 2 minutes of the flagged "low quality" meetings. Did the prospect know why they were there? Did the AE know the context?

**What I'd look for:**
- Is there a gap between what the system knows about the prospect and what the AE knows going into the call?
- Is the AE referencing the outbound email and the prospect's signal in the meeting opener, or are they starting from scratch?

**If this is the problem:** Improve the AE handoff. The Slack notification and Salesforce task should include: (1) the original signal that triggered outreach, (2) the research brief, (3) the email that was sent, and (4) the prospect's reply. The AE should walk in knowing *exactly* why this person is on the call.

### Diagnostic Summary

```
┌─────────────────────────────────────────────────────────────┐
│  SYMPTOM MAP                                                │
│                                                             │
│  45% open rate ──── Subject lines work. Not the problem.    │
│                                                             │
│  3% reply rate ─┬── Messaging misalignment with marketing   │
│                 ├── Personalization too surface-level        │
│                 └── CTA too vague / no clear reason to reply│
│                                                             │
│  "Low quality"  ┬── CTA didn't set meeting expectations     │
│  meetings      ├── AE doesn't have prospect context         │
│                 └── Scoring/filtering criteria too loose     │
│                                                             │
│  AE distrust ───── Natural consequence of the above.        │
│                    Fix the output quality and trust follows. │
│                                                             │
│  Marketing      ── This is both a root cause (messaging     │
│  misalignment      is off) and a collaboration opportunity  │
│                    (fix it together, not in isolation)       │
└─────────────────────────────────────────────────────────────┘
```

### Immediate Actions (This Week)

| Action | Owner | Timeline |
|---|---|---|
| Pull 30 sample emails + compare against marketing positioning | GTME | Day 1 |
| Read Gong recordings of 5 "low quality" meetings | GTME | Day 1 |
| Schedule working session with Revenue Marketing (see Part 2) | GTME | Day 1, session by Day 3 |
| Categorize all replies (positive/negative/neutral) and review CTAs | GTME | Day 2 |
| Audit AE handoff: what context are they actually receiving? | GTME | Day 2 |
| Update `prompt.md` + templates based on findings | GTME | Day 3-4 |
| Share diagnosis + action plan with AE team (see Part 3) | GTME | Day 5 |

---

## Part 2: Slack Message to Revenue Marketing

```
Hey Revenue Marketing team —

I wanted to flag something and get ahead of it. I've been reviewing
the outbound emails our system is generating, and I think there's a
real disconnect between the messaging we're sending and the campaign
positioning you've built for this segment. That's on me to fix.

Rather than trying to guess at the right alignment, I'd love to set
up a working session so we can get this right together. Here's what
I'm thinking:

  What: 45-min working session
  Goal: Align outbound messaging with campaign positioning
  Agenda:
  - You walk me through the positioning framework and key messaging
    pillars for the target segment (15 min)
  - I show you how the outbound system generates emails — the
    templates, the AI personalization layer, and where messaging
    decisions get made (10 min)
  - Together we update the messaging guidelines (prompt.md) and
    outreach templates to match campaign positioning (20 min)

The tangible output would be updated outbound templates and AI
prompts that reflect your positioning. And going forward, I'd want
to build a lightweight review cadence — maybe a monthly async check
where I share sample outbound emails and you flag anything that's
drifted.

Does later this week work? Happy to work around your schedules.
```

**Why this message works:**
- **Takes ownership immediately** ("That's on me to fix") — doesn't blame the system or make excuses
- **Proposes a specific session**, not a vague "let's chat" — agenda, time, and tangible output are defined
- **Shows the system, doesn't just describe it** — inviting them to see how the AI generates emails demystifies the system and gives them confidence they can influence it
- **Establishes an ongoing cadence** — signals this isn't a one-time fix, it's a permanent feedback loop
- **Respects their expertise** — "You walk me through the positioning" positions them as the authority on messaging strategy

---

## Part 3: Building Trust with the AE Team

### The Core Problem

An AE is doing manual prospecting because they don't trust the system. This is rational behavior — they're 3 weeks into a new model with underwhelming results, and their quota depends on pipeline that this system is supposed to generate. The way to build trust is not to tell them the system works. It's to **show them**, **involve them**, and **fix the problems they're seeing**.

### Strategy: Transparency + Voice + Quick Wins

#### 1. Radical Transparency: Weekly Pipeline Dashboard

Build a weekly Slack digest (automated via n8n) that shows each AE, for their territory:

```
┌─────────────────────────────────────────────────────────────┐
│  OUTBOUND SYSTEM REPORT — Week of Feb 17                    │
│  AE: Sarah Chen | Territory: Enterprise West                │
│                                                             │
│  Emails sent:           147                                 │
│  Open rate:             46%                                 │
│  Reply rate:            4.1%                                │
│  Positive replies:      4                                   │
│  Meetings booked:       2                                   │
│                                                             │
│  Top accounts activated this week:                          │
│  • Acme Corp (intent surge — "digital notarization")        │
│  • BigCo Inc (product signup — trial started Feb 14)        │
│  • Realty Partners (usage spike — 3x volume increase)       │
│                                                             │
│  Meetings on your calendar from outbound:                   │
│  • Tue 2/18 — Jsmith@acme.com (VP Ops)                     │
│    Signal: Intent surge + trial signup                      │
│    Brief: [link to research synthesis]                      │
│  • Thu 2/20 — Jdoe@bigco.com (Dir. Legal)                  │
│    Signal: Feature activation (first notarization)          │
│    Brief: [link to research synthesis]                      │
│                                                             │
│  Pipeline created from outbound (cumulative): $127K         │
└─────────────────────────────────────────────────────────────┘
```

**Why this matters:** The system is no longer a black box. Every AE can see exactly what's happening in their territory, what signals are being acted on, and what pipeline is being generated. If the numbers are bad, they can see that too — and that's fine. Transparency in both directions builds trust faster than only showing highlights.

#### 2. Give AEs a Voice: Structured Feedback Loop

**Post-meeting form (30 seconds, in Salesforce or Slack):**
After every outbound-sourced meeting, the AE answers two questions:
1. Was this meeting qualified? (Yes / Partially / No)
2. One sentence on why (free text)

This data feeds directly back into the system:
- If a specific signal source consistently produces "No" ratings, we tighten the activation criteria for that signal
- If a specific industry or persona consistently produces "Partially," we adjust the messaging angle
- AEs see their feedback resulting in system changes — this is what makes them co-owners instead of bystanders

**Weekly 15-minute standup with the AE team:**
- GTME shares: what changed this week based on AE feedback, what metrics look like, what's planned next
- AEs share: what accounts they want prioritized, what messaging resonated in their conversations, what objections they're hearing
- This is not a status meeting — it's a co-design session. The AE who's doing manual prospecting should be explicitly invited to share what's working in their manual approach so it can be incorporated into the system

#### 3. The Rogue AE: Turn Them Into Your Best Collaborator

The AE doing manual prospecting is not a problem to solve — they're a signal to listen to. They're telling you, through their actions, that the system's output isn't good enough yet for their territory. Here's how to handle it:

**Step 1: Don't fight it.** Telling them to stop manual prospecting will make them distrust the system more.

**Step 2: Learn from it.** Ask them directly:
- "What accounts are you targeting manually? What made you pick them?"
- "What messaging are you using? What's getting replies?"
- "What are you seeing in your territory that the system is missing?"

**Step 3: Incorporate their instincts.** If their manual prospecting is outperforming the system, there's something they know that the system doesn't. Maybe they have a stronger sense of which accounts are a fit. Maybe their messaging is better because they understand their territory. Feed those insights into `prompt.md`, templates, and scoring logic.

**Step 4: Show them the result.** "Hey, I incorporated the messaging angle you've been using for [segment] into the system's templates. Here are the first results — reply rate went from 3% to 6% for that segment."

When the system starts performing as well as their manual effort, they'll stop doing manual prospecting on their own — not because you told them to, but because it's no longer necessary.

#### 4. Quick Wins: Make the First Success Stories Visible

Find the first 2-3 outbound-sourced meetings that converted to real pipeline and make them visible to the entire team:

- Slack post in the AE channel: "Pipeline alert: $45K opportunity created at Acme Corp. This started from an AI-generated outbound email triggered by a 6Sense intent surge on 'digital notarization.' The system identified the VP of Ops, personalized the email around their recent ops hires, and booked the meeting. Full context was passed to [AE name], who is now running the deal."
- Include the actual email that was sent (with prospect info anonymized if needed) so AEs can see the quality

**Why this matters:** One concrete success story does more for trust than any amount of metrics or promises. It proves the model can work. And it gives the AE team a reference point: "This is what good output from the system looks like."

### Trust-Building Timeline

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  WEEK 3 (now):                                              │
│  ├── AEs skeptical. One doing manual prospecting.           │
│  ├── Meeting quality complaints.                            │
│  └── System is a black box to the sales team.               │
│                                                             │
│  WEEK 4 (immediate actions):                                │
│  ├── Fix messaging alignment with Revenue Marketing         │
│  ├── Improve AE handoff (full context in Slack + SF)        │
│  ├── Launch weekly pipeline dashboard per AE                │
│  ├── Launch post-meeting feedback form                      │
│  ├── First 15-min standup with AE team                      │
│  └── Talk to rogue AE — learn from their approach           │
│                                                             │
│  WEEKS 5-6 (building momentum):                             │
│  ├── Reply rates start improving (messaging fixed)          │
│  ├── Meeting quality improves (better CTAs + context)       │
│  ├── AE feedback is visibly changing the system             │
│  ├── First pipeline success stories shared in Slack         │
│  └── Rogue AE's messaging incorporated into system          │
│                                                             │
│  WEEKS 7-8 (earning trust):                                 │
│  ├── AEs see consistent meeting quality improvement         │
│  ├── Dashboard shows clear pipeline attribution             │
│  ├── AEs start requesting specific accounts be prioritized  │
│  ├── (This means they trust the system enough to use it)    │
│  └── Manual prospecting naturally decreases                 │
│                                                             │
│  MONTH 3+ (partnership):                                    │
│  ├── AEs and GTME are co-owners of outbound pipeline        │
│  ├── Feedback loop is automatic and continuous               │
│  ├── System performance is visible and accountable           │
│  └── Nobody questions whether the model works — data shows  │
│       it does                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### The Underlying Principle

The GTME's job is not to defend the system. It's to make the system work *for* the AEs. If the AEs aren't getting good meetings, that's the GTME's problem to fix — not the AEs' problem to accept. The moment the GTME starts treating AE feedback as criticism to deflect rather than signal to act on, trust is lost permanently.

The framing should always be: **"Your pipeline, my system. Tell me what's not working and I'll fix it."**
