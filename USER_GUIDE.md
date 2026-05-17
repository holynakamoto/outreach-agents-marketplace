# User guide — outreach-agents

How to actually get value out of this plugin for BD outreach. Read once before your first session, reference as needed.

---

## Mental model — 60 seconds

This isn't six separate skills. It's **one skill with six agent personas**, and Claude picks which persona to use based on what you describe wanting. Your job is to **describe intent in plain English**, not to remember agent names.

The six agents map to a real BD workflow:

```
Research → Personalization → Revenue (send)
    ↓                            ↓
Meeting Prep              [prospect replies / books]
    ↓                            ↓
[meeting happens]         Deal (CRM update)
    ↓
Deal (CRM update)
    ↓
Omni (ad-hoc queries across all the above)
```

You'll spend most of your time in **Research → Personalization → Revenue**. The others are situational.

---

## Day 0: setup (one-time, ~20 min)

### 1. Fill in product context

Open `~/.claude/plugins/cache/<hash>/outreach-agents/skills/outreach-agents/resources/product-context.md` (or, if you cloned the repo, `plugins/outreach-agents/skills/outreach-agents/resources/product-context.md`).

The template has 10 fields. The four that matter most:

- **Product one-liner** — what dave.io actually is, in one sentence. Banned words (transform, leverage, AI-powered) make you sound like every other vendor. Write it like you'd describe it to a fellow engineer at a bar.
- **What it actually does (3 capabilities)** — concrete, not aspirational. "Triages incidents from PagerDuty using LLM classification with a human checkpoint before any production change" beats "Modernizes incident response."
- **ICP** — industry, size, funding, tech stack signals, titles. Be ruthlessly narrow. "Series B–D B2B SaaS, 100–500 eng, Datadog or New Relic in their stack, hiring SRE" is better than "tech companies."
- **Banned words for outbound** — phrases that signal AI-generated mail. Add to this list every time you spot a new one in the wild.

If product-context.md is blank, every Personalization Agent output will sound generic. The skill specifically refuses to draft outreach when context is generic and will prompt you to fill it in.

### 2. Verify MCP connections

In a fresh Claude Code session, run `/mcp` and confirm these are connected and green:

- **Apollo** — for ICP search
- **Notion** — for prospect/opportunity tracking
- **Gmail** — for inbox queries and thread context
- **Slack** — for sharing deal updates (optional but useful)

If any are red, fix before continuing. Each agent's quality degrades sharply without its tool wiring.

### 3. Set up where prospect work lives in Notion

Create or pick:
- One Notion page for **prospect drops** (lists of new contacts to evaluate)
- One Notion database for **opportunities** (deals progressing through stages)
- Optionally: one Slack channel for **deal updates** (e.g. `#bd-pipeline`)

Paste those URLs into `product-context.md`. The Revenue Agent and Deal Agent both write to these locations.

---

## Day 1: your first three sessions

Run these in order. They build on each other.

### Session A: "Find me 10 AI SRE prospects worth a personalized touch"

In Claude Code, paste:

> Use outreach-agents. Find 10 high-intent AI SRE prospects matching dave.io's ICP. Prefer accounts with one of: recent SRE/Platform job postings, public funding in the last 90 days, or visible incidents on their status page in the last 30 days. Output as a Notion-ready table. Don't draft outreach yet.

The Revenue Agent will:
1. Run Apollo company searches with your ICP filters
2. Apply job-posting intent signals via Apollo's job-postings endpoint
3. WebSearch each top-10 candidate for recent news (90 days)
4. Hand you a table: `Company | Contact | Title | Intent Signal | Personalization Hook`

Review the table. Reject any contacts that don't match your gut sense of fit. Don't skip this — Apollo's data is 80% right, not 100%.

### Session B: "Research the top 3 in depth, then draft personalized outreach"

> Use outreach-agents. Take the top 3 prospects from the list in the Notion page <URL>. For each, run a medium-depth research dossier. Then draft three personalized cold emails per prospect (technical / business / curiosity hooks). I'll pick one per contact before any send.

The skill chains Research → Personalization. Output is a Notion page or chat thread with:

- Per prospect: 1 dossier + 3 email variants
- Each variant cites a specific signal from the dossier
- No buzzword bingo (the SKILL.md has explicit banned phrases)

**Important: review every email before send.** Especially in the first month. The skill is a draft engine, not an autopilot. The whole reason it doesn't auto-send is that one bad message to a high-value prospect costs more than 10 good ones generate.

### Session C: "Prep for tomorrow's call"

> Use outreach-agents. I have a discovery call tomorrow at 2pm with the VP Platform Eng at <Company>. Prep me. Here's our prior thread: <paste Gmail or Notion link>

The Meeting Prep Agent generates a one-pager: who's in the room, where you are with the account, three things that happened in the last 7 days, suggested agenda, disqualifying questions, pre-call ask. Total ~300 words.

Read it 15 min before the call. Skim once more in the 60 seconds before joining.

---

## Day 7+: weekly rhythm

The cadence that actually scales:

| When | What | Time |
|---|---|---|
| **Monday 9 AM** | Run "weekly prospect drop" workflow — 25 new prospects matching ICP, top 5 with full research, drafts ready in Notion | 30 min |
| **Daily before any meeting** | Run Meeting Prep Agent on the day's calendar | 5 min × N meetings |
| **After every call** | Paste transcript → Deal Agent proposes CRM updates → review → apply | 5 min |
| **Friday afternoon** | Omni Agent: "Show me everyone we touched this week who hasn't replied. Which look like fits for a LinkedIn voice note?" | 15 min |
| **Monthly first Friday** | Review product-context.md. Update banned phrases. Refine ICP based on what closed vs. didn't. | 30 min |

Total time investment: ~3 hours/week of human input, generating ~25 high-touch prospects + maintaining pipeline hygiene across ~30 active opportunities.

---

## Tactical playbook: AI SRE buyers specifically

Engineering buyers respond to a narrower set of signals than typical B2B prospects. Tune your usage of the skill accordingly.

### What signals to prioritize

| Signal | Why it works | Where the skill finds it |
|---|---|---|
| **Open SRE/Platform Eng job postings** | They're feeling pain right now; budget is committed | Apollo `apollo_organizations_job_postings` |
| **Recent status page incidents** | Concrete pain you can reference without being creepy | Research Agent WebFetches their status page |
| **Public engineering blog post about incident response** | They've publicly named the problem; you have permission to engage | Research Agent scrapes their eng blog |
| **VP Eng change in last 90 days** | New leader = new budget = receptive to vendor pitches | Apollo enrich + WebSearch on exec moves |
| **Recent Series B/C/D funding** | Infra spend headroom + pressure to scale eng efficiency | Apollo enrich shows funding history |

### What signals to avoid (false positives)

- **Generic "AI" mentions on their blog.** Doesn't mean they're buying AI SRE. Filter against.
- **Listed PagerDuty as a customer logo.** They're a PD competitor or partner, not necessarily a prospect. Disqualify.
- **Hiring "DevOps Engineer."** Often signals a more traditional ops culture, less likely to buy AI SRE. Hiring "Platform Engineer" or "Site Reliability Engineer" is a stronger signal.

### Title taxonomy and what each cares about

The Personalization Agent should adapt to the title:

| Title | Their top concern | The angle that works |
|---|---|---|
| **VP Engineering** | MTTR, on-call burnout cost, eng retention | Business outcome — "cut after-hours pages by 60%" |
| **Director Platform Eng** | Tooling stack coherence, infra cost, eng productivity | Integration depth — "fits into your existing Datadog + Linear flow" |
| **SRE Manager** | Their on-call rotation, runbook coverage, blameless culture | Co-pilot framing — "augments your runbooks; the human checkpoint is the feature, not a limitation" |
| **CTO (smaller co)** | Headcount efficiency, time-to-incident-resolution metrics | ROI calculator — "replaces ~0.5 FTE of after-hours coverage" |
| **CISO** | Audit trail, compliance, change control | Governance framing — "every action logged with human approval before prod write" |

The Personalization Agent's three-variant output usually gives you one variant per persona-fit. Pick the one that matches the recipient's title.

### Cadence that works for engineering buyers

Smartlead is your sequence engine; this skill is your draft engine. Recommended cadence:

```
T+0d   Email 1 (Personalization Agent, technical hook variant) — 50 words
T+3d   LinkedIn connection request, no message
T+4d   Email 2 (follow-up, references your eng blog post or a customer's incident pattern)
T+7d   LinkedIn voice note, 30 sec — re-frame the email's value prop in your voice
T+11d  Email 3 (curiosity hook — "saw <Competitor Z> just had a 2-hour outage. Worth 15 min on how dave.io would have caught that?")
T+18d  Email 4 (break-up — "should I assume not a fit and stop pinging?")
```

Reply rate target: 5-8% across the sequence. Below 3% means your ICP is too broad or product-context.md is too generic. Above 10% means you've nailed it — write down what's working before you forget.

### Subject line patterns that work (and don't)

**Works:**
- Reference a specific artifact: `quick note on your eng blog post`, `re: <Company> status page incident 5/12`
- Mutual connection (if real): `<MutualName> said you might find this useful`
- Short and lowercase: `quick question about your runbook stack`

**Doesn't work:**
- Anything with `[CASE STUDY]` or `[Important]` brackets
- Numbers in the subject line (`5 ways to...`, `Increase MTTR by 40%`)
- Questions that feel ChatGPT-generated (`Are you tired of on-call burnout?`)

---

## Common failure modes

These are the patterns that kill BD pipeline. The skill is designed to resist all of them, but only if you don't override:

| Anti-pattern | Symptom | Fix |
|---|---|---|
| **Spraying generic templates** | Reply rate < 1%, sender reputation tanking | Use Personalization Agent for everyone above $50K ACV potential. Only Smartlead-blast the long tail. |
| **Auto-sending without review** | A bad email lands in a key prospect's inbox; relationship dead | First 30 days: review every send. After that: spot-check 1 in 5. |
| **Skipping Research to "save time"** | Personalization comes out generic because Personalization Agent has no signals to reference | Research is the highest-ROI step. 5 min of Research saves 50 wasted prospect touches. |
| **Updating product-context.md once and forgetting it** | Drift between your messaging and what's actually closing deals | Monthly review. Add wins, kill losers, refresh trigger events. |
| **Treating Omni Agent as Google** | Hallucinated "data" because it inferred from limited context | Omni only answers from connected MCPs. If it has to guess, it says so. If it doesn't say so, push back. |
| **Letting Deal Agent auto-write to Notion** | Garbage in your CRM | Default is review-before-apply. Don't change to autopilot. |
| **Cold-emailing from `@dave.io`** | One bad week kills your primary domain reputation forever | Use Smartlead with secondary domains (`trydave.io`, etc.). See README for setup. |

---

## How to iterate the skill

The skill is a living document. After 30 days of use:

1. **Capture every weird output** — when the Personalization Agent writes something cringey, screenshot it and note WHY. Add the offending phrase to the banned list in `product-context.md`.
2. **Track reply rates by variant type** — if technical-hook variants consistently outperform business-outcome variants, edit SKILL.md to put technical first.
3. **Track signals → close rate** — if "open SRE jobs" prospects close at 3× the rate of "recent funding" prospects, weight Apollo searches more heavily on hiring signals.
4. **Push improvements upstream** — `cd ~/Tools/outreach-agents-marketplace && git commit -am "Bump banned phrases" && git push`. Teammates pull updates with `/plugin marketplace update`.

If you want a version that captures lessons across multiple users in your team, set up a shared Notion page titled `outreach-agents — what we've learned`. Whoever updates the skill checks the Notion page first.

---

## Integration with Smartlead (volume layer)

This skill is the **high-touch top 20%**. For the long-tail 80% (where personalization-per-prospect isn't ROI-positive), Smartlead is the volume engine.

Recommended split:

| Lane | Volume | Tool | Personalization |
|---|---|---|---|
| **High-touch** | 25–50 prospects/week | outreach-agents (this skill) | Full Research → 3 variants per prospect |
| **Mid-touch** | 200–400 prospects/week | Smartlead with templates + dynamic fields | Light — first sentence personalized, rest templated |
| **Pure volume** | 1,000+ prospects/week | Smartlead pure templates | None — wide net |

Use the skill to draft the Smartlead templates: ask it to "write a Smartlead T1 template for SRE Managers at Series B SaaS that's personalized via the `{{first_name}}` and `{{company}}` merge fields, with the first sentence referencing the company's hiring page if dynamically discoverable." Then Smartlead's enrichment fills the variables at send time.

---

## When NOT to use this skill

- **Inbound replies / customer success** — different motion, different skill.
- **Drafting a generic blog post / landing page** — use `copywriting` or `content-creator` skills.
- **One-off email to a known contact** — overkill; just write it.
- **CCPA/privacy outreach** — use the eraser pipeline at `~/Tools/eraser`, not this.

---

## Getting help

- Plugin source: https://github.com/holynakamoto/outreach-agents-marketplace
- File issues for bugs or feature requests on that repo
- Internal: maintain a `outreach-agents — what we've learned` Notion page where your team documents wins, losses, and product-context.md tweaks
