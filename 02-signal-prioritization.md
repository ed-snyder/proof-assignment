# Signal-Based Prioritization & Enrichment Logic

## Signals Selected

**Signal 1: 6Sense Intent Surges** — Third-party intent data showing which accounts are actively researching solutions in Proof's category. This is the strongest "why now" signal because it captures buying behavior *before* a prospect ever visits your site or fills out a form.

**Signal 2: Product Usage Data (Redshift)** — First-party behavioral data from Proof's self-serve / PLG motion: signups, feature activation, transaction volume, and drop-offs. This is the highest-fidelity signal because it's based on what people *actually do with the product*, not inferred interest.

Together these cover the full intent spectrum: 6Sense captures accounts that are *thinking about* a solution, product data captures people who are *already using* one.

---

## Signal 1: 6Sense Intent Surges — Integration Walkthrough

### The Core Problem

6Sense is account-level, not contact-level. When 6Sense says "Acme Corp is surging on digital notarization," it doesn't tell you *who* at Acme to email. The integration has to solve this account-to-contact gap.

### Architecture

```
6Sense API → n8n → 6Sense People Search API → Clay (enrichment) → Salesforce (store) → Outreach (activate)
```

### Step-by-Step Technical Walkthrough

#### Step 1: Pull Surging Accounts from 6Sense

**n8n node:** Schedule Trigger (daily, 6am) → HTTP Request node

6Sense doesn't expose a single "give me all surging accounts" bulk endpoint. There are two viable approaches:

**Approach A: Segment-based pull (recommended)**
Configure a segment in the 6Sense UI using the Audience Builder. The segment filters for:
- Buying stage = "Decision" or "Purchase"
- Intent topics include Proof-relevant keywords (e.g., "digital notarization," "remote online closing," "eSignature," "identity verification")
- Profile fit = "Strong" or "Moderate"

Then pull segment membership via the 6Sense API. The segment ID is referenced in API responses, and segment membership can be exported or queried. This offloads the filtering logic to 6Sense's scoring engine, which is purpose-built for this.

**Approach B: Per-account scoring lookup**
Hit the Company Identification API per account domain:

```
GET https://epsilon.6sense.com/v3/company/details
Headers:
  Authorization: Token <40_CHAR_API_TOKEN>
```

**Response includes:**
```json
{
  "company": {
    "name": "Acme Corp",
    "domain": "acme.com",
    "industry": "Real Estate",
    "employee_range": "1,000 - 4,999",
    "revenue_range": "$100M - $250M"
  },
  "scores": [
    {
      "product": "Proof",
      "company_intent_score": 78,
      "company_buying_stage": "Decision",
      "company_profile_score": 85,
      "company_profile_fit": "Strong"
    }
  ],
  "segments": {
    "ids": [4713, 28237],
    "names": ["ICP Accounts", "High Intent Surge"]
  }
}
```

This approach works but requires iterating over your account universe. At 50K accounts, that's 50K API calls — feasible given 6Sense's 10 queries/sec rate limit on the People Search API, but the segment-based approach is more efficient.

**n8n implementation:** The HTTP Request node hits the 6Sense API, a Code node parses the response, and an IF node filters to only accounts where `company_buying_stage` is "Decision" or "Purchase" AND `company_intent_score` > 60.

#### Step 2: Resolve Accounts to Contacts

**n8n node:** HTTP Request node → 6Sense People Search API

For each surging account, we need to find the right contacts. The 6Sense People Search API handles this:

```
POST https://api.6sense.com/v2/search/people
Headers:
  Authorization: Token <API_TOKEN>
  Content-Type: application/json

Body:
{
  "domain": ["acme.com"],
  "level": ["Director", "VP", "C-Level"],
  "function": ["Operations", "Legal", "Compliance", "Product"],
  "pageNo": 1,
  "pageSize": 10
}
```

**Response includes:** email, phone, mobile, job title, seniority, job function, LinkedIn URL, location.

**n8n implementation:** A Loop node iterates over each surging account. For each, the HTTP Request node calls the People Search API with the account's domain and our target persona filters (level + function). A Code node selects the top 2-3 contacts per account based on seniority and relevance.

**Rate limit note:** 6Sense People Search API allows 10 queries/sec. For a daily batch of ~100 surging accounts, this completes in ~10 seconds.

#### Step 3: Enrich Contacts via Clay

**n8n node:** HTTP Request node → Clay API

The 6Sense People Search gives us basic contact info, but Clay adds depth: recent LinkedIn activity, company news, tech stack, hiring signals, etc. This enrichment is what makes the downstream personalization work.

```
POST https://api.clay.com/v1/people/enrich
Body:
{
  "email": "jsmith@acme.com",
  "company_domain": "acme.com"
}
```

#### Step 4: Write to Salesforce

**n8n node:** Salesforce node (native n8n integration)

**Account-level field mapping:**

| 6Sense Field | Salesforce Field | Type |
|---|---|---|
| `company_buying_stage` | `X6Sense_Buying_Stage__c` | Picklist |
| `company_intent_score` | `X6Sense_Intent_Score__c` | Number |
| `company_profile_fit` | `X6Sense_Profile_Fit__c` | Picklist |
| `scores[].product` | `X6Sense_Intent_Topic__c` | Text |
| `segments.names` | `X6Sense_Segments__c` | Multi-select Picklist |
| Last surge date | `X6Sense_Last_Surge_Date__c` | Date |

**Contact-level field mapping:**

| Source | Salesforce Field | Type |
|---|---|---|
| 6Sense People Search: email | `Email` | Standard |
| 6Sense People Search: title | `Title` | Standard |
| 6Sense People Search: LinkedIn | `LinkedIn_URL__c` | URL |
| Clay: tech stack | `Tech_Stack__c` | Long Text |
| Clay: recent news | `Company_News__c` | Long Text |
| Signal source tag | `Signal_Source__c` | Picklist |
| Signal strength score | `Signal_Strength__c` | Number |

n8n's Salesforce node uses the Salesforce REST API natively. For accounts, it upserts on `Domain` (custom external ID field). For contacts, it upserts on `Email`.

#### Step 5: Activate in Outreach

**n8n node:** HTTP Request node → Outreach API (OAuth 2.0)

Outreach uses **JSON:API 1.0** format and OAuth 2.0. The activation flow:

**5a. Upsert the prospect:**

Outreach doesn't have a native upsert — you lookup first, then create or update:

```
GET https://api.outreach.io/api/v2/prospects?filter[emails]=jsmith@acme.com
Headers:
  Authorization: Bearer <ACCESS_TOKEN>
  Content-Type: application/vnd.api+json
```

If not found, create:

```
POST https://api.outreach.io/api/v2/prospects
Body:
{
  "data": {
    "type": "prospect",
    "attributes": {
      "emails": ["jsmith@acme.com"],
      "firstName": "John",
      "lastName": "Smith",
      "title": "VP Operations",
      "company": "Acme Corp",
      "custom1": "6sense_intent_surge",
      "custom2": "Decision",
      "custom3": "78"
    }
  }
}
```

Custom fields (`custom1` through `custom150`) carry the signal metadata into Outreach where it can be used in template variables and reporting.

**5b. Add to sequence:**

```
POST https://api.outreach.io/api/v2/sequenceStates
Body:
{
  "data": {
    "type": "sequenceState",
    "relationships": {
      "prospect": { "data": { "type": "prospect", "id": 12345 } },
      "sequence": { "data": { "type": "sequence", "id": 42 } },
      "mailbox": { "data": { "type": "mailbox", "id": 7 } }
    }
  }
}
```

All three relationships — `prospect`, `sequence`, and `mailbox` — are required. The sequence ID maps to the signal-appropriate sequence (intent surge contacts get a different sequence than product signups). The mailbox ID determines which sending identity is used.

**Rate limit note:** Outreach allows 10,000 requests/hour. For a daily batch of ~200-300 contacts, this is well within limits.

### Messaging Within This Flow: Hybrid Approach

**Static layer:** Outreach templates define the email structure. Created via `POST /api/v2/templates`:

```json
{
  "data": {
    "type": "template",
    "attributes": {
      "name": "Intent Surge - First Touch",
      "subject": "{{prospect.custom4}}, quick question about {{prospect.custom5}}",
      "bodyHtml": "<p>Hi {{prospect.custom4}},</p><p>{{prospect.custom6}}</p><p>{{prospect.custom7}}</p>"
    }
  }
}
```

**Dynamic layer:** The AI-generated content from the personalized first-touch workflow (Assessment 1) gets written into Outreach custom fields when we upsert the prospect:
- `custom4` = transformed first name / conversational greeting
- `custom5` = conversational company name
- `custom6` = personalized opening (the "why now" from hypothesis generation)
- `custom7` = CTA

This means the template *structure* is static and controlled by the GTME, but the *content* filling each slot is dynamically generated per contact. The template acts as a guardrail — it ensures every email follows the right structure — while AI handles the personalization.

**Feedback layer:** Insights from Gong mining (Section 3 below) update which templates get used for which segments, and which objections get preemptively addressed in the messaging framework.

---

## Signal 2: Product Usage Data — Integration Walkthrough

### Architecture

```
Redshift (product DB) → n8n → Salesforce (match + store) → Scoring logic → Outreach (activate)
```

### Step-by-Step Technical Walkthrough

#### Step 1: Query Product Data

**n8n node:** Schedule Trigger (daily, 6am) → Postgres node (Redshift is Postgres-compatible)

```sql
-- New pro signups (last 24 hours)
SELECT user_id, email, company_name, signup_date, plan_type
FROM users
WHERE plan_type = 'pro_trial'
  AND signup_date >= CURRENT_DATE - INTERVAL '1 day'

UNION ALL

-- Feature activation (completed first notarization)
SELECT u.user_id, u.email, u.company_name, e.event_date, 'feature_activation' as signal
FROM users u
JOIN events e ON u.user_id = e.user_id
WHERE e.event_type = 'notarization_completed'
  AND e.event_date >= CURRENT_DATE - INTERVAL '1 day'
  AND e.event_count = 1  -- first time

UNION ALL

-- Usage spike (transaction volume 2x+ week-over-week)
SELECT u.user_id, u.email, u.company_name, CURRENT_DATE as signal_date, 'usage_spike' as signal
FROM users u
JOIN (
  SELECT user_id,
         SUM(CASE WHEN event_date >= CURRENT_DATE - INTERVAL '7 days' THEN 1 ELSE 0 END) as this_week,
         SUM(CASE WHEN event_date >= CURRENT_DATE - INTERVAL '14 days'
                   AND event_date < CURRENT_DATE - INTERVAL '7 days' THEN 1 ELSE 0 END) as last_week
  FROM events
  WHERE event_type = 'transaction'
  GROUP BY user_id
) t ON u.user_id = t.user_id
WHERE t.this_week >= t.last_week * 2
  AND t.last_week > 0

UNION ALL

-- Drop-off (signed up 7+ days ago, no activity in last 7 days)
SELECT u.user_id, u.email, u.company_name, u.signup_date, 'drop_off' as signal
FROM users u
LEFT JOIN events e ON u.user_id = e.user_id
  AND e.event_date >= CURRENT_DATE - INTERVAL '7 days'
WHERE u.signup_date <= CURRENT_DATE - INTERVAL '7 days'
  AND e.user_id IS NULL
  AND u.plan_type = 'pro_trial'
```

#### Step 2: Filter Against Salesforce

**n8n node:** Salesforce node → SOQL query

Before activating any product-sourced contact, we check:
- Not already a paying customer (`Opportunity.StageName != 'Closed Won'`)
- Not already in an active Outreach sequence
- Not in a "do not contact" segment

```sql
SELECT Id, Email, AccountId, Account.Name
FROM Contact
WHERE Email IN ('jsmith@acme.com', 'jdoe@bigco.com')
  AND Account.Id NOT IN (
    SELECT AccountId FROM Opportunity WHERE StageName = 'Closed Won'
  )
```

**n8n implementation:** A Code node takes the Redshift results and builds a batch SOQL query. The Salesforce node executes it. A Merge node (inner join on email) keeps only contacts that pass the filter.

#### Step 3: Score and Tag

**n8n node:** Code node

Each contact gets a `signal_source` tag and `signal_strength` score:

| Product Signal | Signal Source Tag | Strength Logic |
|---|---|---|
| New pro signup | `product_signup` | Base: 60. +10 if company size > 100, +10 if in target industry |
| First notarization | `feature_activation` | Base: 75. +10 if multiple team members invited |
| Usage spike (2x+) | `usage_spike` | Base: 80. +5 per additional multiplier |
| Drop-off (no activity 7d) | `drop_off` | Base: 50. -10 per additional week of inactivity |

These scores are relative — they determine priority order within the daily batch, not absolute thresholds. A usage spike contact gets activated before a drop-off contact.

#### Step 4: Write to Salesforce + Activate in Outreach

Same mechanics as Signal 1 (Steps 4-5 above). The key difference is the signal metadata written to custom fields, which drives template selection in Outreach.

### Messaging Within This Flow

Product signals demand **different messaging than intent signals** because the prospect already has hands-on experience with Proof:

| Signal | Messaging Angle | Template Strategy |
|---|---|---|
| New signup | "Welcome + here's how to get the most out of your trial" | Helpful, not salesy. Reference their specific use case if known from signup form |
| Feature activation | "You just completed your first [X] — here's what teams like yours do next" | Expand the conversation from individual use to team/enterprise |
| Usage spike | "Your volume is growing — let's make sure you're set up for scale" | Enterprise features, pricing conversation, dedicated support |
| Drop-off | "Noticed you haven't had a chance to [complete onboarding step] — can I help?" | Friction removal, offer a walkthrough, low-pressure |

These are **distinct Outreach sequences** (not just different first emails) because the follow-up cadence and escalation logic differ by signal type.

---

## Mining Closed-Lost Data & Gong Recordings

This is the feedback loop that makes the entire system smarter over time. It's a **batch analysis job** — not a real-time workflow — that runs monthly and outputs structured intelligence the GTME uses to update targeting rules, messaging templates, and `prompt.md`.

### The Technical Challenge

Gong has no "give me calls from closed-lost deals" filter. You can't query Gong by deal outcome. Instead, you have to:

1. Query Salesforce for closed-lost Opportunity IDs
2. Pull call data from Gong with CRM context
3. Match calls to closed-lost opps client-side
4. Pull transcripts for matched calls
5. Feed transcripts to Claude for analysis

This is best handled as a Python script, not an n8n workflow, because it involves cross-system joins, pagination, rate limiting, and LLM batch processing.

### Python Script: Gong Closed-Lost Analysis Pipeline

```python
"""
gong_closed_lost_analysis.py

Pulls closed-lost deal data from Salesforce, matches to Gong call recordings,
extracts transcripts, and uses Claude to identify objection patterns,
segments to deprioritize, and messaging recommendations.

Run monthly: python gong_closed_lost_analysis.py
Output: closed_lost_intelligence.json
"""

import requests
import json
import time
import base64
from datetime import datetime, timedelta
from anthropic import Anthropic

# ─── Configuration ─────────────────────────────────────────────
SALESFORCE_BASE_URL = "https://yourinstance.salesforce.com"
SALESFORCE_ACCESS_TOKEN = "your_sf_token"  # OAuth token

GONG_ACCESS_KEY = "your_gong_access_key"
GONG_ACCESS_SECRET = "your_gong_access_secret"
GONG_BASE_URL = "https://api.gong.io/v2"

ANTHROPIC_API_KEY = "your_anthropic_key"

LOOKBACK_MONTHS = 12
OUTPUT_FILE = "closed_lost_intelligence.json"


# ─── Step 1: Query Salesforce for Closed-Lost Opportunities ────
def get_closed_lost_opps():
    """
    Pull closed-lost opportunities from the last 12 months via Salesforce SOQL.
    Returns list of {opp_id, account_name, industry, employee_count,
                     close_date, amount, loss_reason, competitor}.
    """
    soql = f"""
        SELECT Id, Name, AccountId, Account.Name, Account.Industry,
               Account.NumberOfEmployees, CloseDate, Amount,
               Loss_Reason__c, Competitor__c, OwnerId, Owner.Name
        FROM Opportunity
        WHERE StageName = 'Closed Lost'
          AND CloseDate >= LAST_N_MONTHS:{LOOKBACK_MONTHS}
        ORDER BY CloseDate DESC
    """
    headers = {
        "Authorization": f"Bearer {SALESFORCE_ACCESS_TOKEN}",
        "Content-Type": "application/json"
    }
    resp = requests.get(
        f"{SALESFORCE_BASE_URL}/services/data/v59.0/query",
        params={"q": soql},
        headers=headers
    )
    resp.raise_for_status()
    records = resp.json()["records"]

    opps = []
    for r in records:
        opps.append({
            "opp_id": r["Id"],
            "opp_name": r["Name"],
            "account_id": r["AccountId"],
            "account_name": r["Account"]["Name"],
            "industry": r["Account"].get("Industry", "Unknown"),
            "employee_count": r["Account"].get("NumberOfEmployees", 0),
            "close_date": r["CloseDate"],
            "amount": r.get("Amount", 0),
            "loss_reason": r.get("Loss_Reason__c", ""),
            "competitor": r.get("Competitor__c", ""),
            "owner_name": r["Owner"]["Name"]
        })

    print(f"Found {len(opps)} closed-lost opportunities")
    return opps


# ─── Step 2: Pull Gong Calls With CRM Context ─────────────────
def get_gong_calls_for_period(from_date, to_date):
    """
    Pull all calls from Gong within a date range using /v2/calls/extensive.
    This endpoint returns CRM context (opportunity IDs) linked to each call.

    Gong rate limit: 3 requests/sec, 10,000/day.
    Pagination: cursor-based, 100 records per page.
    """
    auth = base64.b64encode(
        f"{GONG_ACCESS_KEY}:{GONG_ACCESS_SECRET}".encode()
    ).decode()

    headers = {
        "Authorization": f"Basic {auth}",
        "Content-Type": "application/json"
    }

    all_calls = []
    cursor = None

    while True:
        body = {
            "filter": {
                "fromDateTime": from_date,
                "toDateTime": to_date
            },
            "contentSelector": {
                "context": "Extended",
                "exposedFields": {
                    "content": {
                        "topics": True,
                        "trackers": True,
                        "brief": True,
                        "highlights": True
                    }
                }
            }
        }
        if cursor:
            body["cursor"] = cursor

        resp = requests.post(
            f"{GONG_BASE_URL}/calls/extensive",
            headers=headers,
            json=body
        )
        resp.raise_for_status()
        data = resp.json()

        calls = data.get("calls", [])
        all_calls.extend(calls)

        cursor = data.get("records", {}).get("cursor")
        if not cursor:
            break

        # Respect rate limit: 3 requests/sec
        time.sleep(0.35)

    print(f"Pulled {len(all_calls)} total Gong calls")
    return all_calls


def match_calls_to_closed_lost(gong_calls, closed_lost_opps):
    """
    Match Gong calls to closed-lost opportunities by CRM opportunity ID.
    Gong's /v2/calls/extensive response includes CRM context with opp IDs.
    """
    # Build lookup set of closed-lost opp IDs
    lost_opp_ids = {opp["opp_id"] for opp in closed_lost_opps}
    opp_lookup = {opp["opp_id"]: opp for opp in closed_lost_opps}

    matched_calls = []
    for call in gong_calls:
        # CRM context is nested in the call's context field
        crm_context = call.get("context", [])
        for ctx in crm_context:
            # The CRM object references include opportunity IDs
            objects = ctx.get("objects", [])
            for obj in objects:
                if obj.get("objectType") == "Opportunity":
                    opp_id = obj.get("objectId")
                    if opp_id in lost_opp_ids:
                        matched_calls.append({
                            "call_id": call["metaData"]["id"],
                            "call_title": call["metaData"].get("title", ""),
                            "call_date": call["metaData"].get("started", ""),
                            "duration_seconds": call["metaData"].get(
                                "duration", 0
                            ),
                            "opp_id": opp_id,
                            "opp_data": opp_lookup[opp_id],
                            "topics": call.get("content", {}).get(
                                "topics", []
                            ),
                            "trackers": call.get("content", {}).get(
                                "trackers", []
                            ),
                            "highlights": call.get("content", {}).get(
                                "highlights", []
                            )
                        })

    print(f"Matched {len(matched_calls)} calls to closed-lost deals")
    return matched_calls


# ─── Step 3: Pull Transcripts for Matched Calls ───────────────
def get_transcripts(call_ids):
    """
    Pull transcripts for a list of call IDs using POST /v2/calls/transcript.

    Gong returns transcripts as speaker monologues, each containing
    an array of sentences with millisecond timestamps.

    Batches in groups of 100 (API limit per request).
    """
    auth = base64.b64encode(
        f"{GONG_ACCESS_KEY}:{GONG_ACCESS_SECRET}".encode()
    ).decode()

    headers = {
        "Authorization": f"Basic {auth}",
        "Content-Type": "application/json"
    }

    all_transcripts = {}

    # Batch call IDs in groups of 100
    for i in range(0, len(call_ids), 100):
        batch = call_ids[i:i + 100]
        cursor = None

        while True:
            body = {
                "filter": {
                    "callIds": batch
                }
            }
            if cursor:
                body["cursor"] = cursor

            resp = requests.post(
                f"{GONG_BASE_URL}/calls/transcript",
                headers=headers,
                json=body
            )
            resp.raise_for_status()
            data = resp.json()

            for t in data.get("callTranscripts", []):
                # Flatten transcript into readable text
                text_parts = []
                for monologue in t.get("transcript", []):
                    speaker = monologue.get("speakerId", "Unknown")
                    sentences = " ".join(
                        s["text"] for s in monologue.get("sentences", [])
                    )
                    text_parts.append(f"[Speaker {speaker}]: {sentences}")
                all_transcripts[t["callId"]] = "\n".join(text_parts)

            cursor = data.get("records", {}).get("cursor")
            if not cursor:
                break

            time.sleep(0.35)

    print(f"Retrieved {len(all_transcripts)} transcripts")
    return all_transcripts


# ─── Step 4: Analyze With Claude ───────────────────────────────
def analyze_closed_lost_batch(matched_calls, transcripts):
    """
    Feed call transcripts + deal metadata to Claude for structured analysis.

    Processes in batches of 5 calls per Claude request to stay within
    context limits while giving the model enough data to spot patterns.

    Output per batch: objections, pain points, competitor mentions,
    segments to deprioritize, messaging recommendations.
    """
    client = Anthropic(api_key=ANTHROPIC_API_KEY)

    all_analyses = []

    for i in range(0, len(matched_calls), 5):
        batch = matched_calls[i:i + 5]

        # Build context for this batch
        batch_context = []
        for call in batch:
            transcript = transcripts.get(call["call_id"], "No transcript")
            # Truncate long transcripts to ~4000 chars to fit context
            if len(transcript) > 4000:
                transcript = transcript[:4000] + "\n[...truncated]"

            batch_context.append(f"""
--- DEAL: {call['opp_data']['opp_name']} ---
Account: {call['opp_data']['account_name']}
Industry: {call['opp_data']['industry']}
Size: {call['opp_data']['employee_count']} employees
Deal Value: ${call['opp_data']['amount']}
Loss Reason (CRM): {call['opp_data']['loss_reason']}
Competitor: {call['opp_data']['competitor']}
Close Date: {call['opp_data']['close_date']}
Gong Topics: {', '.join(call.get('topics', []))}
Gong Trackers: {', '.join(call.get('trackers', []))}

TRANSCRIPT:
{transcript}
""")

        prompt = f"""Analyze these {len(batch)} closed-lost deal calls from Proof, a digital notarization and identity verification platform. For each call, and then in aggregate, extract:

1. **Primary Objections**: What specific objections did the prospect raise? Categorize them (pricing, feature gap, competitor preference, timing, internal priority, regulatory concern, etc.)

2. **Pain Points Mentioned**: What problems was the prospect trying to solve? Were they a good fit for Proof's solution?

3. **Competitor Mentions**: Which competitors were mentioned? What was the prospect's perception of them vs Proof?

4. **Segment Signals**: Based on the industry, company size, and use case — does this look like a segment Proof should deprioritize? Why or why not?

5. **Messaging Gaps**: Was there anything in the sales conversation that suggests our outbound messaging could have set better expectations or addressed concerns earlier?

Return your analysis as structured JSON with this schema:
{{
  "per_deal": [
    {{
      "account_name": "...",
      "industry": "...",
      "primary_objection": "...",
      "objection_category": "...",
      "pain_points": ["..."],
      "competitors_mentioned": ["..."],
      "should_deprioritize": true/false,
      "deprioritize_reason": "...",
      "messaging_recommendation": "..."
    }}
  ],
  "aggregate_patterns": {{
    "top_objections": [{{"objection": "...", "frequency": N, "category": "..."}}],
    "segments_to_avoid": [{{"segment": "...", "reason": "..."}}],
    "segments_to_double_down": [{{"segment": "...", "reason": "..."}}],
    "messaging_changes": ["..."],
    "competitor_positioning": [{{"competitor": "...", "perception": "...", "counter_strategy": "..."}}]
  }}
}}

CALLS:
{''.join(batch_context)}"""

        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            messages=[{"role": "user", "content": prompt}]
        )

        try:
            analysis = json.loads(response.content[0].text)
            all_analyses.append(analysis)
        except json.JSONDecodeError:
            # If Claude doesn't return clean JSON, store raw text
            all_analyses.append({"raw_response": response.content[0].text})

        print(f"Analyzed batch {i // 5 + 1}/{(len(matched_calls) + 4) // 5}")

    return all_analyses


# ─── Step 5: Synthesize Into Intelligence Brief ───────────────
def synthesize_intelligence(all_analyses):
    """
    Combine batch analyses into a single intelligence brief.
    This is the output the GTME uses to update targeting and messaging.
    """
    client = Anthropic(api_key=ANTHROPIC_API_KEY)

    response = client.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=4096,
        messages=[{"role": "user", "content": f"""You are synthesizing the results of a closed-lost deal analysis for Proof (digital notarization platform). Below are analyses from multiple batches of lost deals.

Produce a single, actionable intelligence brief with:

1. **Top 5 Objection Patterns** — ranked by frequency, with recommended messaging responses for each

2. **Segments to Deprioritize** — industries, company sizes, or use cases where Proof consistently loses. For each, state the reason and recommended action (suppress entirely, or adjust messaging)

3. **Segments to Prioritize** — where Proof consistently wins or where losses were close/winnable

4. **Messaging Playbook Updates** — specific changes to make to outbound messaging based on what we learned. Frame as updates to prompt.md and outreach templates

5. **Competitor Intelligence** — what prospects say about competitors, and how to position against them

BATCH ANALYSES:
{json.dumps(all_analyses, indent=2)}"""}]
    )

    return response.content[0].text


# ─── Main Execution ────────────────────────────────────────────
def main():
    print("=" * 60)
    print("GONG CLOSED-LOST ANALYSIS PIPELINE")
    print(f"Lookback: {LOOKBACK_MONTHS} months")
    print("=" * 60)

    # Step 1: Get closed-lost opps from Salesforce
    opps = get_closed_lost_opps()
    if not opps:
        print("No closed-lost opportunities found. Exiting.")
        return

    # Step 2: Pull Gong calls and match to closed-lost opps
    from_date = (
        datetime.now() - timedelta(days=LOOKBACK_MONTHS * 30)
    ).strftime("%Y-%m-%dT00:00:00Z")
    to_date = datetime.now().strftime("%Y-%m-%dT23:59:59Z")

    gong_calls = get_gong_calls_for_period(from_date, to_date)
    matched = match_calls_to_closed_lost(gong_calls, opps)

    if not matched:
        print("No Gong calls matched to closed-lost deals. Exiting.")
        return

    # Step 3: Pull transcripts
    call_ids = [c["call_id"] for c in matched]
    transcripts = get_transcripts(call_ids)

    # Step 4: Analyze with Claude
    analyses = analyze_closed_lost_batch(matched, transcripts)

    # Step 5: Synthesize intelligence brief
    brief = synthesize_intelligence(analyses)

    # Save outputs
    output = {
        "generated_at": datetime.now().isoformat(),
        "lookback_months": LOOKBACK_MONTHS,
        "total_closed_lost_opps": len(opps),
        "total_matched_calls": len(matched),
        "total_transcripts": len(transcripts),
        "batch_analyses": analyses,
        "intelligence_brief": brief
    }

    with open(OUTPUT_FILE, "w") as f:
        json.dump(output, f, indent=2)

    print(f"\nIntelligence brief saved to {OUTPUT_FILE}")
    print("\n" + "=" * 60)
    print("INTELLIGENCE BRIEF")
    print("=" * 60)
    print(brief)


if __name__ == "__main__":
    main()
```

### How the Feedback Loop Works

The script outputs `closed_lost_intelligence.json`. The GTME reviews this and takes three actions:

**1. Update suppression rules in n8n**
If the analysis reveals that a segment (e.g., companies < 50 employees in healthcare) consistently churns, the GTME adds an IF node filter in the n8n workflow (from Assessment 1, Step 0) to exclude that segment before enrichment even begins. No wasted Clay credits, no wasted emails.

**2. Update `prompt.md` with objection preemption**
If the analysis reveals a recurring objection (e.g., "we need on-prem deployment"), the GTME adds this to `prompt.md` so the research synthesis agent knows to:
- Check if the prospect's industry/size typically raises this objection
- If yes, preemptively address it in the "Why Proof" hypothesis
- If the objection is a true dealbreaker (Proof can't solve it), flag the contact for suppression

**3. Update Outreach templates**
If messaging gaps are identified (e.g., "prospects didn't understand our pricing model"), the GTME updates the relevant Outreach template to address this in the first-touch email.

### Refresh Cadence

| Action | Frequency | Why |
|---|---|---|
| Full Gong analysis pipeline run | Monthly | 150 closed-lost opps / 12 months ≈ ~12 new data points/month. Monthly gives enough new data to spot trends without over-reacting |
| GTME reviews intelligence brief | Monthly (day after pipeline run) | Human judgment on which recommendations to implement |
| Suppression list updates | Monthly (after review) | Too frequent = over-filtering; too infrequent = wasting outreach on bad segments |
| `prompt.md` updates | Monthly + ad-hoc | Monthly from Gong insights; ad-hoc from quality gate hard-fail patterns |
| Outreach template updates | Monthly + ad-hoc | Same as above |
| Lightweight check for new closed-lost opps | Weekly | Quick SOQL query to flag any high-value losses worth immediate attention (e.g., a $100K+ deal lost to a competitor — don't wait for the monthly run) |

---

## Defining and Measuring Success

### Core Metrics

| Metric | What It Measures | Target | Review Cadence |
|---|---|---|---|
| **Signal-to-activation rate** | % of surfaced signals that result in a contact being activated in Outreach | > 70% | Weekly |
| **Signal-to-reply rate** | % of activated contacts that reply (by signal source) | > 8% overall; segment by signal source | Weekly |
| **Signal-to-meeting rate** | % of activated contacts that book a meeting | > 3% | Weekly |
| **Signal-to-pipeline rate** | % of activated contacts that generate qualified pipeline ($) | > 1% | Monthly |
| **AE acceptance rate** | % of meetings AEs accept as "qualified" | > 80% | Weekly |
| **Enrichment coverage** | % of contacts with complete enrichment data | > 85% | Daily |
| **Suppression accuracy** | % of suppressed contacts that would have been non-responsive (validated via periodic sampling) | > 90% | Monthly |
| **Time-to-activation** | Time from signal firing to email sent | < 24 hours | Daily |

### Signal Source Comparison (the most important view)

| Signal Source | Reply Rate | Meeting Rate | Pipeline $ | AE Acceptance |
|---|---|---|---|---|
| 6Sense intent surge | Track | Track | Track | Track |
| Product signup | Track | Track | Track | Track |
| Feature activation | Track | Track | Track | Track |
| Usage spike | Track | Track | Track | Track |
| Drop-off | Track | Track | Track | Track |

This table, populated over time, tells you **which signals are actually worth acting on**. If "drop-off" contacts consistently produce 1% reply rates while "usage spike" contacts produce 15%, you reallocate effort accordingly.

### Iteration Cadence

**Daily (automated monitoring):**
- n8n workflow execution success/failure rates
- Enrichment fill rates
- Signal volume (are we getting expected volume from each source?)
- Quality gate pass/fail rates

**Weekly (GTME reviews):**
- Reply rate and meeting rate by signal source
- AE feedback on meeting quality
- Hard fail patterns from quality gate
- Implement tactical changes: adjust scoring thresholds, tweak template copy, update IF node filters

**Monthly (strategic review):**
- Run Gong closed-lost analysis pipeline
- Review intelligence brief with AE team and Revenue Marketing
- Update suppression lists, `prompt.md`, and Outreach templates
- Decide whether to add/remove signal sources based on performance data
- Pipeline attribution: which signals drove the most pipeline $ this month?

---

## Research Methodology & Sources

The technical integration details in this document were researched by querying the official API documentation for each platform. Below is how each was investigated and the primary sources used.

### Gong API
- **Authentication:** Basic auth (access key + secret) or OAuth 2.0. Scopes are granular (e.g., `api:calls:read:transcript`, `api:calls:read:extensive`)
- **Key finding:** There is no "filter by deal stage" parameter on any Gong endpoint. Calls must be pulled via `/v2/calls/extensive` (which includes CRM context), then matched client-side to closed-lost opportunity IDs from Salesforce. This is the primary architectural constraint that shaped the Python script approach.
- **Rate limits:** 3 API calls/sec, 10,000/day (company-wide). Cursor-based pagination, 100 records/page.
- **Sources:**
  - [Gong Official API Docs — What the API Provides](https://help.gong.io/docs/what-the-gong-api-provides)
  - [Gong Official — Create an OAuth App](https://help.gong.io/docs/create-an-app-for-gong)
  - [Gong Official — Receive API Access](https://help.gong.io/docs/receive-access-to-the-api)
  - [Gong Community — Retrieve Call Transcript via Python](https://visioneers.gong.io/data-in-gong-71/retrieve-call-transcript-through-api-using-python-1158)
  - [Gong Community — Fetching Calls by CRM Deal](https://visioneers.gong.io/developers-79/fetching-all-calls-associated-with-a-crm-deal-through-the-api-1332)
  - [Gong API Beginners Guide (Postman)](https://www.postman.com/growment/gong-meetup/documentation/yuikwaq/gong-api-beginners-guide)

### 6Sense API
- **Authentication:** Token-based header (`Authorization: Token <40_CHAR_KEY>`)
- **Key finding:** 6Sense is fundamentally account-level. The Company Identification API returns intent scores, buying stages, and segment membership per account. To get contacts, you must use the separate People Search API (`POST /v2/search/people`) which supports filters for domain, job title, level, and function (max 10 values per filter).
- **Buying stages:** Target → Awareness → Consideration → Decision → Purchase. Exposed as `company_buying_stage` in the scores array.
- **Rate limits:** 10 queries/sec (People Search), 20/sec (Enrichment). API calls consume credits from purchased allocation.
- **Sources:**
  - [6Sense API Portal](https://api.6sense.com/docs) (requires authentication)
  - [6Sense Support — API Overview](https://support.6sense.com/docs/6sense-api-overview)
  - [6Sense Support — APIs](https://support.6sense.com/docs/apis)
  - [6Sense Support — API Credits & Tokens](https://support.6sense.com/docs/api-credits-api-tokens)
  - [6Sense Support — Predictive Buying Stages](https://support.6sense.com/docs/predictive-buying-stages)
  - [6Sense Support — Scores Overview](https://support.6sense.com/docs/6sense-scores-overview)
  - [6Sense Support — Bombora Surge Data FAQ](https://support.6sense.com/docs/faq-using-bombora-company-surge-data-within-6sense)

### Outreach API
- **Authentication:** OAuth 2.0 (Authorization Code Grant) exclusively — no simple API keys. Access tokens expire in 2 hours; refresh tokens in 14 days.
- **Key finding:** Outreach follows JSON:API 1.0 spec. There is no native upsert endpoint — you must lookup by email (`GET /api/v2/prospects?filter[emails]=...`), then create or update. Adding a prospect to a sequence requires creating a `sequenceState` resource with three required relationships: prospect, sequence, and mailbox.
- **Personalized emails:** Outreach doesn't have a "send this email now" endpoint. Emails are sent through sequences. Templates use `{{prospect.fieldName}}` variables, so AI-generated content is written into prospect custom fields (`custom1`–`custom150`) which the template then references.
- **Rate limits:** 10,000 requests/hour per user.
- **Sources:**
  - [Outreach Developer Docs — OAuth](https://developers.outreach.io/api/oauth/)
  - [Outreach Developer Docs — Getting Started](https://developers.outreach.io/api/getting-started/)
  - [Outreach Developer Docs — Common Patterns](https://developers.outreach.io/api/common-patterns/)
  - [Outreach Developer Docs — Webhooks](https://developers.outreach.io/api/webhooks/)
  - [Outreach API Reference — Prospects](https://developers.outreach.io/api/reference/tag/Prospect/)
  - [Outreach API Reference — Sequence States](https://developers.outreach.io/api/reference/tag/Sequence-State/)
  - [Outreach API Reference — Templates](https://developers.outreach.io/api/reference/tag/Template/)
  - [Outreach Support — How to Access APIs](https://support.outreach.io/hc/en-us/articles/9986965571867-How-to-Access-Outreach-APIs)

### Salesforce API
- Standard REST API (`/services/data/v59.0/query`) with SOQL queries. OAuth 2.0 authentication. Used for closed-lost opportunity queries and contact/account record management. Well-documented at [developer.salesforce.com](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/).
