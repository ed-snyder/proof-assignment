# Question 4: Performance Iteration & Cross-Functional Collaboration

## Diagnosing the Root Causes

The 45% open rate tells us subject lines work and deliverability is healthy. That's not the problem. The problem is downstream: the email body isn't earning replies, and when it does earn a meeting, the meeting quality is poor. These aren't four separate issues. They're symptoms of the same root causes.

### What I'd Investigate First

**Messaging misalignment (most likely primary cause).** Revenue Marketing already told us the outbound copy doesn't match their campaign positioning. If marketing is positioning around one value prop (say, "eliminate notarization fraud risk") and our outbound emails are leading with a different one ("speed up your closing workflow"), prospects who've seen Proof's brand get a dissonant first touch. It doesn't land because it doesn't match the narrative they've already been exposed to.

I'd pull Revenue Marketing's campaign positioning document for the target segment and 20-30 actual emails the system sent (not the templates, the rendered output). Then do a side-by-side comparison: where do the messaging angles, language, and value props diverge? Are we leading with a different value prop? Using different terminology for the same concepts? Is the tone mismatched?

**Personalization depth.** High open rate plus low reply rate often means the email got attention but didn't earn a response. I'd read those same 20-30 sent emails and check whether the personalization is actually specific. "I noticed you just posted 3 ops roles, sounds like you're scaling your closing workflow" is good. "As a leader in real estate, you know closings are complex" is generic filler wearing a personalization costume. I'd also check quality gate logs. If the pass rate is 95%+, the gate might be too lenient.

**CTA and expectation setting.** If the CTA is vague ("would you be open to a quick chat?"), two things happen: most people don't reply because there's no specific reason to, and the few who say yes show up not knowing why they're on the call. That's exactly what the AEs are reporting. I'd categorize all replies from the past 3 weeks and for the meetings AEs flagged as "low quality," pull the original email. What was the CTA? What expectation did it set?

**AE context gap.** The "low quality" feedback might partly be an information problem on the AE side. I'd check what the AE actually sees before a meeting. Is the research brief making it to them through the Slack notification and Salesforce task? I'd pull Gong recordings of the first 2 minutes of the flagged meetings. Did the AE reference the outbound email and the prospect's signal, or did they start from scratch?

### Immediate Actions

Day 1: pull 30 sample emails and compare against marketing positioning. Listen to Gong recordings of 5 "low quality" meetings. Schedule the working session with Revenue Marketing. Day 2: categorize all replies and review CTAs. Audit what context AEs are actually receiving before meetings. Days 3-4: update prompt.md and templates based on findings. Day 5: share the diagnosis and action plan with the AE team.

## Slack Message to Revenue Marketing

> Hey Revenue Marketing team,
>
> I wanted to flag something and get ahead of it. I've been reviewing the outbound emails our system is generating, and I think there's a real disconnect between the messaging we're sending and the campaign positioning you've built for this segment. That's on me to fix.
>
> Rather than trying to guess at the right alignment, I'd love to set up a working session so we can get this right together. Here's what I'm thinking:
>
> **What:** 45-min working session
> **Goal:** Align outbound messaging with campaign positioning
> **Agenda:**
> - You walk me through the positioning framework and key messaging pillars for the target segment (15 min)
> - I show you how the outbound system generates emails, the templates, the AI personalization layer, and where messaging decisions get made (10 min)
> - Together we update the messaging guidelines and outreach templates to match campaign positioning (20 min)
>
> The tangible output would be updated outbound templates and AI prompts that reflect your positioning. And going forward, I'd want to build a lightweight review cadence, maybe a monthly async check where I share sample outbound emails and you flag anything that's drifted.
>
> Does later this week work? Happy to work around your schedules.

The message takes ownership immediately rather than blaming the system. It proposes a specific session with a defined agenda and tangible output instead of a vague "let's chat." It invites them to see how the system works, which demystifies it and gives them confidence they can influence it. And it establishes an ongoing cadence so this isn't a one-time fix.

## Building Trust with the AE Team

An AE doing manual prospecting on the side is rational behavior. They're three weeks into a new model with underwhelming results and their quota depends on pipeline this system is supposed to generate. The way to build trust is not to tell them the system works. It's to show them, involve them, and fix the problems they're seeing.

### Transparency: Weekly Pipeline Dashboard

Build a weekly Slack digest (automated via n8n) showing each AE, for their territory: emails sent, open rate, reply rate, positive replies, meetings booked, top accounts activated that week with signal context, upcoming meetings with links to the research brief, and cumulative pipeline created from outbound. The system is no longer a black box. Every AE can see exactly what's happening in their territory. If the numbers are bad, they see that too, and that's fine. Transparency in both directions builds trust faster than only showing highlights.

### Voice: Structured Feedback Loop

After every outbound-sourced meeting, the AE answers two questions in a 30-second form: was this meeting qualified (yes/partially/no) and one sentence on why. This data feeds directly back into the system. If a specific signal source consistently produces "no" ratings, we tighten activation criteria. If a persona consistently produces "partially," we adjust the messaging angle. AEs see their feedback resulting in system changes. That's what makes them co-owners instead of bystanders.

A weekly 15-minute standup with the AE team keeps this running. The GTME shares what changed this week based on AE feedback, what metrics look like, and what's planned next. AEs share what accounts they want prioritized, what messaging resonated in their conversations, and what objections they're hearing. This isn't a status meeting. It's a co-design session.

### The AE Doing Manual Prospecting

This person isn't a problem to solve. They're a signal to listen to. They're telling you, through their actions, that the system's output isn't good enough yet for their territory.

Step one: don't fight it. Telling them to stop will make them distrust the system more. Step two: learn from it. Ask them directly what accounts they're targeting, what messaging they're using, what they're seeing that the system is missing. Step three: incorporate their instincts. If their manual prospecting is outperforming the system, there's something they know that the system doesn't. Feed those insights into prompt.md, templates, and scoring logic. Step four: show them the result. "I incorporated the messaging angle you've been using for that segment into the system's templates. Here are the first results, reply rate went from 3% to 6% for that segment."

When the system starts performing as well as their manual effort, they'll stop doing manual prospecting on their own. Not because you told them to, but because it's no longer necessary.

### Quick Wins: Make Success Visible

Find the first 2-3 outbound-sourced meetings that converted to real pipeline and make them visible to the entire team. A Slack post in the AE channel: "Pipeline alert: $45K opportunity created at Acme Corp. This started from an AI-generated outbound email triggered by a 6Sense intent surge. The system identified the VP of Ops, personalized the email around their recent ops hires, and booked the meeting." Include the actual email so AEs can see the quality. One concrete success story does more for trust than any amount of metrics or promises.

### The Underlying Principle

The GTME's job is not to defend the system. It's to make the system work for the AEs. If they're not getting good meetings, that's the GTME's problem to fix, not the AEs' problem to accept. The framing should always be: "Your pipeline, my system. Tell me what's not working and I'll fix it."
