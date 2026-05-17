---
name: outreach-agents
description: Multi-agent business-development orchestrator mirroring Outreach.ai's AI Agents suite (Omni, Revenue, Research, Meeting Prep, Deal, Personalization, Orchestration). Use for cold outreach campaigns, prospect research, account-level intel gathering, meeting prep briefings, CRM updates from call transcripts, and personalized message generation. Routes to the correct sub-agent based on user intent. Wired to Apollo, Gmail, Notion, Slack, and WebSearch.
metadata:
  model: sonnet
---

## When to use this skill

Invoke when the user asks to:
- Find prospects matching an ICP and generate outreach
- Research an account, person, or company in depth
- Prep for an upcoming customer meeting
- Update CRM/opportunity records from a meeting or email thread
- Personalize a cold or follow-up message
- Ask a natural-language question about pipeline, accounts, or activity ("how many SRE leads did we touch this week?")
- Orchestrate a multi-step BD workflow

## When NOT to use this skill

- Pure copywriting unrelated to a specific prospect — use `sales-automator` instead.
- Privacy/CCPA outreach — different stack entirely (use the eraser pipeline at `~/Tools/eraser`).
- Internal communication, customer success, support — out of scope.

## Routing — pick the agent that matches user intent

| User says / wants | Agent | Section below |
|---|---|---|
| "Who should I reach out to?" / "Find prospects matching…" / "Build a list of…" | **Revenue Agent** | §1 |
| "Tell me about <company/person>" / "Research <X>" / "What's <company> up to?" | **Research Agent** | §2 |
| "Prep me for the <X> meeting" / "Brief on <account> before tomorrow" | **Meeting Prep Agent** | §3 |
| "Update the opportunity" / "Log this call" / "What changed for <deal>?" | **Deal Agent** | §4 |
| "Write a personalized email to…" / "Make this not sound like a template" | **Personalization Agent** | §5 |
| "How many… / show me… / when did… (about pipeline data)" | **Omni Agent** | §6 |
| Multi-step workflow chaining 2+ of the above | **Orchestration** | §7 |

If the intent could match multiple agents, run them in this order: Research → Personalization → Revenue (send). Use TaskCreate to track sub-tasks when chaining.

---

## Product context (FILL THIS IN before first use)

Read `resources/product-context.md` for the active product positioning. If empty or generic, ask the user before drafting any outreach. Every agent below assumes accurate product context — sending generic messaging is the failure mode this skill exists to prevent.

---

## §1 — Revenue Agent

**Role:** identify high-intent accounts matching the ICP, find decision-maker contacts, generate the first-touch outreach for each.

**Inputs (ask if missing):**
- ICP definition: industry, company size band, geo, tech stack signals, funding stage, hiring signals
- Quota: how many qualified prospects to surface this run (default 25; cap at 100 per run to keep outputs reviewable)
- Channels: email-only, email + LinkedIn, or email + LinkedIn + voice note

**Tools to use:**
- `mcp__claude_ai_Apollo_io__apollo_mixed_people_api_search` for contact search
- `mcp__claude_ai_Apollo_io__apollo_mixed_companies_search` for account search
- `mcp__claude_ai_Apollo_io__apollo_organizations_job_postings` for intent signals (companies hiring SREs = AI SRE buyers)
- `WebSearch` for fresh news/funding/exec hires per account
- `mcp__claude_ai_Notion__notion-create-pages` to log the list

**Workflow:**
1. Parse ICP into Apollo search filters. Confirm with user before executing.
2. Run `apollo_mixed_companies_search` first to find target accounts. Filter for intent signals: recent funding, hiring SRE/Platform/DevOps roles, technology fingerprints.
3. For each account, pull 1–3 contacts via `apollo_mixed_people_api_search`. Prioritize: VP Eng, Director Platform Eng, SRE Manager, Head of DevOps.
4. For top 10 accounts, run a quick `WebSearch` for recent news (last 90 days) — funding, exec hires, product launches, outages. This is the personalization hook.
5. Hand off to Personalization Agent (§5) for the actual message draft per contact. Do not write templates here.
6. Log the prospect list to Notion: account, contact, role, intent signal, personalization hook, status=`queued`.

**Output:** a table with columns: `Company | Contact | Title | Intent Signal | Personalization Hook | Status`. Show in chat AND log to Notion.

**Anti-patterns:**
- Don't send the same template to 25 contacts — that's what Smartlead is for, not Claude. Use this skill for the high-touch top 20% where personalization > volume.
- Don't auto-send. Always pause for human review of the list before any send.

---

## §2 — Research Agent

**Role:** deep research on a single account, contact, or topic to surface signals worth acting on.

**Inputs:**
- Subject: company name OR person LinkedIn URL OR account ID
- Depth: `light` (5-min surface scan) / `medium` (15 min, last-quarter context) / `deep` (full 30-min dossier)

**Tools:**
- `WebSearch` for recent news, blog posts, press releases, podcast appearances
- `WebFetch` for the company's own pages (about, careers, engineering blog, security pages)
- `mcp__claude_ai_Apollo_io__apollo_organizations_enrich` for firmographic data
- `mcp__claude_ai_Apollo_io__apollo_organizations_job_postings` for hiring intent
- `mcp__firecrawl-mcp__firecrawl_scrape` if a target site blocks WebFetch

**Workflow (medium depth example):**
1. Firmographic baseline via Apollo enrich: HQ, headcount, revenue est, tech stack, funding rounds.
2. Last 90 days news via WebSearch — quote source + date for every claim. Categorize: funding / exec moves / product / customer wins / incidents / press.
3. Engineering signals: scrape their engineering blog (last 6 posts), careers page (open SRE/Platform roles), status page (recent incidents — direct AI SRE talking point).
4. Decision-maker layer: top 3 contacts by relevance to the buying committee for AI SRE. Pull title, tenure, last-job, public posts/talks in the past year.
5. Synthesize: 3 specific "openings" — concrete reasons to reach out *now*, each tied to a primary source.

**Output structure:**

```
# <Company> dossier — <date>

## Firmographic snapshot
- ...

## Recent signals (last 90d)
- [date] event — source

## Engineering posture
- Tech stack: ...
- Recent posts: ...
- Open SRE/Platform roles: N

## Buying committee
| Name | Title | Why they care |

## Three openings (ranked by recency × relevance)
1. ...
2. ...
3. ...
```

**Anti-patterns:**
- Don't fabricate. If WebSearch finds nothing for a claim, say so. The Research Agent's value is *signal extraction*, not narrative generation.
- Don't stop at firmographics. Apollo data alone is what every other BDR has.

---

## §3 — Meeting Prep Agent

**Role:** generate a tight pre-meeting brief 1–24 hours before a scheduled customer call.

**Inputs:**
- Attendee list (names, emails, or LinkedIn URLs)
- Account name (or derive from email domains)
- Meeting type: discovery / demo / technical deep dive / negotiation / executive sponsor
- Prior touchpoints (paste any prior email thread or Notion deal page)

**Tools:**
- Research Agent (§2) for any unfamiliar attendees — call as a sub-step
- `mcp__claude_ai_Notion__notion-search` for prior thread on the account
- `mcp__claude_ai_Gmail__search_threads` to pull prior email history
- `WebSearch` for last-7-day news about the account (don't repeat full Research dossier)

**Output structure (1-pager):**

```
# Meeting brief: <Account> · <date/time>

## Who's in the room
- <Name>, <title> — <one-line signal from their public posts/recent role>

## Where we are with this account
- First touch: <date>, channel
- Open items from last conversation
- Stated pain points

## Three things to know going in (last 7 days)
- ...

## Suggested agenda + talking points
1. ...
2. ...

## Disqualifying questions
- ...  (questions whose answers would tell us this isn't a fit, so we don't waste the cycle)

## Pre-call ask
- ...  (what do we want them to send / confirm before the call)
```

Brief should be ≤300 words. Skip sections that add no signal.

---

## §4 — Deal Agent

**Role:** propose CRM-style updates to an opportunity based on a meeting transcript, recording summary, or email thread.

**Inputs:**
- Source material: paste a Fathom/Granola/Otter transcript, a Gemini call summary, or a Gmail thread
- Account/deal identifier in Notion (or `--new` to create one)
- Current stage if known

**Tools:**
- `mcp__claude_ai_Notion__notion-fetch` to load the current opportunity record
- `mcp__claude_ai_Notion__notion-update-page` to apply changes (with explicit user approval first)
- `mcp__claude_ai_Slack__authenticate` + Slack send for "promote summary to the team" flow

**Workflow:**
1. Read the source material; extract: confirmed pain points, technical constraints, evaluation criteria, competing tools mentioned, decision timeline, blockers, named stakeholders not yet engaged.
2. Diff against current Notion record. Surface a numbered "proposed updates" list — each item is one CRM field change with the source quote that justifies it.
3. Always show the proposed updates to the user before applying. Never auto-write to Notion without confirmation. Outreach.ai's Deal Agent has both modes; this skill defaults to the safer "review then apply" mode.
4. After approval, apply via `notion-update-page` and post a 5-line summary to the relevant Slack channel via Slack MCP if the user asks.

**Output structure:**

```
# Deal update proposal — <Account>

## What changed in this conversation
- ...

## Proposed field updates (review before apply)
1. **Stage**: Discovery → Technical Eval — quote: "<source>"
2. **Confirmed pains**: + "incident response time during EU on-call gaps" — quote: "<source>"
3. **Tech stack notes**: + "uses Datadog + Rootly, evaluating both" — quote: "<source>"
4. ...

## Slack-ready summary (if user wants to share)
> 3 lines max, technical specifics, no marketing language
```

---

## §5 — Personalization Agent

**Role:** take research output + product context + a touchpoint slot in a sequence, return one message that doesn't sound like AI slop or a template.

**Inputs:**
- Research dossier or specific opening (from §2)
- Sequence position: T1 cold / T2 follow-up / T3 break-up / reply-to-objection
- Channel: email / LinkedIn / voice note script
- Length budget: ≤60 words for T1, ≤40 for follow-ups

**Tools:** none — pure synthesis from the inputs.

**Rules of the agent:**
1. **Open with the signal, not the offer.** First sentence references something specific from research (a post they wrote, an incident on their status page, a job posting). Never "Hope you're doing well."
2. **One concrete value claim, not three.** Pick the single sharpest dave.io capability that matches the signal. Skip the rest.
3. **Soft CTA, not a calendar link in T1.** "Worth a 15-min walkthrough?" beats "Book a time here." The Calendly link goes in T2 or after a reply.
4. **No buzzwords list:** transform, leverage, synergy, AI-powered, revolutionize, game-changing, unlock, holistic. Engineering buyers see these as red flags.
5. **Match their writing register.** If their engineering blog is dry/technical, you write dry/technical. If they post memes, you can be lighter.
6. **Three drafts.** Output three short variants with slightly different angles (technical / business-outcome / curiosity-bait). User picks one.

**Output structure:**

```
# Three variants — <Recipient> @ <Account>

## Variant A — Technical hook
Subject: <≤6 words>
Body: <≤60 words>

## Variant B — Business outcome hook
Subject: <≤6 words>
Body: <≤60 words>

## Variant C — Curiosity hook
Subject: <≤6 words>
Body: <≤60 words>

## Signal used
<one line — which specific research artifact each variant referenced>
```

---

## §6 — Omni Agent

**Role:** natural-language Q&A across the user's BD data sources, with the ability to take actions.

**Inputs:**
- A question in plain English about pipeline, activity, accounts, or contacts

**Tools (in order of preference):**
- `mcp__claude_ai_Notion__notion-search` for opportunity/account records the user has logged
- `mcp__claude_ai_Gmail__search_threads` for email-thread-based queries
- `mcp__claude_ai_Slack__authenticate` + Slack search for team-channel context
- `mcp__claude_ai_Apollo_io__*` for cold pipeline data not yet in Notion

**Workflow:**
1. Parse the question into (a) what data source to query, (b) what filter/aggregation, (c) whether an action follows.
2. Execute the read. Show counts, list, or summary as appropriate.
3. If the user's question implies an action ("…and send them a follow-up"), confirm before doing it. Hand off to Revenue/Personalization for sends.

**Example interactions:**
- "How many AI SRE prospects did I email last week?" → Gmail search by date range + filter on Smartlead sequence label.
- "Show me the top 5 accounts in our pipeline by recency of touch" → Notion query.
- "Which prospects haven't replied after 3 emails?" → Gmail thread analysis.

This agent is intentionally the most flexible. If you can't answer from available data, say so explicitly — don't fabricate.

---

## §7 — Orchestration (Agent Studio equivalent)

**Role:** chain agents into a workflow when the user describes a multi-step task.

**Common pre-built workflows:**

**A) "Weekly prospect drop"** — for a steady BD cadence
1. Revenue Agent: 25 new prospects matching ICP (filter: not already in Notion)
2. Research Agent: light-depth dossier per top-5 of those
3. Personalization Agent: 3 variants per top-5
4. Output: a Notion page titled `Prospect drop — <date>` with the table + drafts ready for human review and Smartlead upload

**B) "Pre-meeting sweep"** — daily 7 AM
1. Pull tomorrow's calendar (manual paste for now, or Google Calendar MCP if connected)
2. For each customer meeting: Meeting Prep Agent generates the brief
3. Deliver all briefs in one Notion page, sorted by meeting time

**C) "Post-call CRM hygiene"** — after a recorded call
1. User pastes transcript + opportunity ID
2. Deal Agent proposes updates
3. On approval: applies updates to Notion + posts Slack summary

**D) "Reply triage"** — when a prospect replies
1. User pastes the reply
2. Classify intent: interested / not now / not interested / unsubscribe / question
3. If interested: draft a calendar-link follow-up. If question: draft a substantive answer with a soft CTA. If not now: schedule a check-in in 60d.
4. Always show the draft for review before send.

**Use TaskCreate** to track multi-step orchestration runs. Mark each sub-agent's work as a separate task.

---

## Failure modes to actively avoid

| Pattern | Why bad | Do instead |
|---|---|---|
| Auto-sending without human review | One bad message tanks domain reputation | Always pause for confirmation on first 10 outputs of any new sequence |
| Generic "AI-generated" personalization | Engineering buyers can smell template-with-mail-merge | Personalization Agent's first sentence must reference an artifact, not a fact |
| Apollo email blast > 50/day | Reputation damage on dave.io variants | Smartlead with rotated mailboxes, 30–50/inbox/day |
| Fabricating signals when research is thin | Burns trust on reply | Research Agent must cite source URL per claim; if nothing found, say "no recent public signals" |
| Skipping the research step "to save time" | Cold spray for engineering buyers gets ~0.5% reply | Research is the non-negotiable step; volume is downstream |

## Tools this skill relies on

Required:
- `WebSearch`, `WebFetch`
- `mcp__claude_ai_Apollo_io__*` (Apollo MCP)
- `mcp__claude_ai_Notion__*` (Notion MCP)
- `mcp__claude_ai_Gmail__*` (Gmail MCP)
- `TaskCreate`, `TaskUpdate` for orchestration tracking

Optional but improves quality:
- `mcp__claude_ai_Slack__*` for team-channel context and posting
- `mcp__firecrawl-mcp__*` for sites that block WebFetch
- `mcp__claude_ai_Google_Calendar__*` for meeting prep auto-trigger

## Initial setup checklist

Before first run, the user should:
1. Fill in `resources/product-context.md` with the actual product positioning. Generic positioning = generic outreach.
2. Confirm Apollo, Notion, Gmail MCPs are authenticated (`/mcp` to check status).
3. Decide where to log prospect lists in Notion — create a parent page or database, paste the URL into `resources/product-context.md`.
4. Decide which Slack channel (if any) gets deal-update summaries.

If any of these are missing, ask the user to set them up before proceeding.
