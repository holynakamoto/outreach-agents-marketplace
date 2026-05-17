# outreach-agents-marketplace

Internal Claude Code plugin marketplace for the dave.io BD team.

## What's inside

One plugin: **`outreach-agents`** — a multi-agent BD orchestrator mirroring Outreach.ai's AI Agents suite (Revenue, Research, Meeting Prep, Deal, Personalization, Omni) wired to Apollo, Notion, Gmail, Slack, and WebSearch.

## Install (for teammates)

In any Claude Code session:

```
/plugin marketplace add <owner>/outreach-agents-marketplace
/plugin install outreach-agents@outreach-agents-marketplace
```

Replace `<owner>` with the actual GitHub owner you push to (e.g., `dave-io`, `nickmoore`).

When the plugin is updated upstream, teammates run:

```
/plugin marketplace update
```

## Before using the skill, every install needs

Fill in `plugins/outreach-agents/skills/outreach-agents/resources/product-context.md` with dave.io positioning, ICP, value props, and trigger events. Without this, every Revenue/Personalization output will be generic.

If you want one canonical product-context shared by the whole team, edit it once in this repo and push — teammates get the latest on `/plugin marketplace update`. (If individuals want to override for their own use case, they can edit their local cache at `~/.claude/plugins/cache/...` — those overrides survive until the next update.)

## Repo layout

```
.
├── .claude-plugin/
│   └── marketplace.json        # catalog
├── plugins/
│   └── outreach-agents/
│       ├── .claude-plugin/
│       │   └── plugin.json     # plugin manifest
│       └── skills/
│           └── outreach-agents/
│               ├── SKILL.md
│               └── resources/
│                   └── product-context.md
└── README.md
```

## Versioning

Bump `version` in `plugins/outreach-agents/.claude-plugin/plugin.json` before pushing breaking changes. Semver-ish:
- patch (0.1.0 → 0.1.1) — typo fixes, copy edits in agent prompts
- minor (0.1.0 → 0.2.0) — new agent, new workflow, new tool wiring
- major (0.1.0 → 1.0.0) — breaking schema changes to product-context.md or sub-agent inputs
