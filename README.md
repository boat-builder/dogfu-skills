# dogfu-skills

A plugin marketplace for **Claude Cowork**. It currently ships one plugin:

| Plugin | What it does |
| :-- | :-- |
| **`dogfu`** | Sales toolkit powered by the `dogfu` CLI. Five skills: **lead-research** (**Scout → checkpoint → Deep dive → CRM** — research a sales target cheaply, brief the user for the **pursue-or-drop** call, then enrich pursued leads; the user's decision (never the skill) puts each lead in Close — Qualified or Bad Fit — with structured company attributes), **lead-touch** (work an existing lead in Close from cold outreach through a closed deal — the **cold** flow: pull what's due, record touches the BDR sent, mark replies, move statuses, report the funnel; and the **warm** flow: run discovery, open/advance/close an **opportunity** (Discovery → Trial → Proposal → Won/Lost), set the next step, surface deals due or stalled, and forecast the pipeline), **crm-cleanup** (read-only health audit — find the leads and tasks that fell out of the flow, each with the exact fix command), **first-audit** (an outside-in **SEO/AEO** audit of a prospect domain from public data — public-site crawl + keyword/competitor/traffic intelligence + live Google/AI answers — published as a single-page report whose URL is recorded on the lead; also uses the **Bluesnake MCP**), and **berlin-theme** (Berlin's brand style guide — apply the editorial cream/paper house style to any sales collateral you design: pages, one-pagers, posters, emails. No CLI; theme only). |

The two CRM-operating skills form one lifecycle: **lead-research** researches a lead and records the user's decision →
**lead-touch** works it from there — the cold flow (`references/cold-outreach.md`) runs the
outreach cadence until the lead replies (→ *Connected*), and the warm flow
(`references/deals.md`) works the live deal (a Close opportunity) to a close. **crm-cleanup**
audits all of it read-only. The opportunity support the warm
flow uses is shipped in the `dogfu` CLI; it needs a Close opportunity pipeline and the `Engaged`
lead status set up once by a Close admin.

**first-audit** sits outside that CRM lifecycle — it's a standalone sales deliverable. It needs
the **Bluesnake MCP** (a local crawler/auditor, not bundled) for the site crawl, and a
`dogfu report publish` command to push the assembled report to the public bucket — that publish
command is being added to the CLI. It reuses the existing `dogfu` `seo` / `google` / `chatgpt`
groups for all data, and writes only the published audit URL back to Close (`crm note`).

## Layout

```
dogfu-skills/
├── .claude-plugin/
│   └── marketplace.json          # marketplace catalog
└── dogfu/                         # the dogfu plugin (source directory)
    ├── .claude-plugin/
    │   └── plugin.json            # plugin manifest
    ├── .mcp.json                  # shared dogfu MCP server configuration
    └── skills/
        ├── lead-research/         # the lead-research skill (scout → checkpoint → deep dive → CRM)
        │   ├── SKILL.md
        │   ├── README.md
        │   └── references/
        ├── lead-touch/            # the lead-touch skill (CRM ops — cold cadence + live deals)
        │   ├── SKILL.md
        │   ├── README.md
        │   └── references/        # cold-outreach (cadence flow), deals (opportunity flow)
        ├── crm-cleanup/           # the crm-cleanup skill (read-only CRM health audit)
        │   ├── SKILL.md
        │   └── README.md
        ├── first-audit/          # the first-audit skill (outside-in SEO/AEO prospect audit)
        │   ├── SKILL.md
        │   ├── README.md
        │   └── references/        # data-sources, bluesnake, report, design-guide
        └── berlin-theme/          # the berlin-theme skill (brand style guide, no CLI)
            ├── SKILL.md
            └── README.md
```

Skills are auto-discovered from the plugin's `skills/` directory — they are not listed
in the manifest.

## Install in Cowork

In Cowork, open **Customize → Plugins**, add this repository as a marketplace, then
install the **Dogfu** plugin:

- **From a local path** — point the marketplace at this folder. No git remote required.
- **From git** — point it at this repo's URL once it's pushed somewhere reachable.

Once installed, the skills trigger automatically: **lead-research** when you give Claude a
sales target ("research this lead", "is this company a fit", a bare LinkedIn/company
link, etc.), **lead-touch** when you want to work an existing lead — cold outreach ("who do I
follow up with today", "I sent the LinkedIn DM to X", "mark this lead replied",
"what's our funnel look like") or a live deal ("a lead replied and
we had an intro call", "open a deal for X", "move this deal to trial", "which deals need action
today", "what's our pipeline / forecast", "an inbound lead reached out"), **crm-cleanup**
for a read-only health pass ("audit the CRM", "what's broken in Close", "find leads stuck in the
flow"), **first-audit** when you explicitly ask for an outside-in prospect audit ("run a first
audit on <domain>", "do an SEO/AEO audit for this prospect", "build and publish the audit report
for <company>"), and **berlin-theme** when you ask Claude to design a Berlin-branded asset ("make
this landing page on-brand", "build a sales one-pager", "design a social poster in our brand").

## Requirement: the dogfu MCP + CLI

**The plugin bundles the dogfu MCP server** — a remote,
OAuth-authenticated MCP at
`https://backend.agentberlin.ai/dogfu/mcp`, declared in `dogfu/.mcp.json`. In clients that
honor plugin-declared MCP servers, installing the plugin registers it automatically; approve
it and authenticate once, and the skill's `get_setup_instructions` flow is available.

> **Cowork note (verify):** Cowork's documented path for remote OAuth MCPs is its own
> connectors UI (an admin adds the server, members authenticate), so a plugin-bundled OAuth
> MCP may or may not be picked up automatically. If it isn't, add the same URL
> (`https://backend.agentberlin.ai/dogfu/mcp`) via Cowork connectors instead — and the
> `dogfu/.mcp.json` declaration can be reverted with no effect on the rest of the plugin.

The plugin does **not** bundle the `dogfu` CLI. `dogfu` is a published CLI (`pip install
dogfu`) that you install and authenticate through the dogfu MCP: run its
`get_setup_instructions` tool and follow it (install the CLI, then `dogfu configure --otp
<OTP> --title "…"`). CRM (Close) calls use the Close API key you set once in the admin
**Console → CRM Integration**. There's no directory to mount and no `.env`. Without the dogfu
CLI configured, the skill can reason about a target but can't pull data or write to the CRM.

### Extra requirements for `first-audit`

`first-audit` uses the same dogfu MCP + CLI as the rest, plus two things the other skills
don't:

- **The Bluesnake MCP** — a local crawler/auditor that produces the site-crawl + on-page
  (technical/AEO) layer of the audit. It is **not bundled** by this plugin; configure it as a
  separate MCP server (user or project scope), the same way the original Agent Berlin plugin
  did. Without it, the keyword/competitor/answer-engine sections still work, but the crawl and
  on-page findings can't be produced.
- **A `dogfu report publish` command** to push the assembled single-page report to the public
  report bucket and return its URL. This publish path is **being added to the `dogfu` CLI**;
  until it ships, the skill assembles the report locally but can't publish it (it'll say so
  rather than fabricate a URL). The skill builds the report against the vendored design guide
  in `first-audit/references/design-guide.md` (no live design endpoint needed).

## Editing

Edit the markdown under each skill's folder — `dogfu/skills/lead-research/`,
`dogfu/skills/lead-touch/`, `dogfu/skills/crm-cleanup/`, `dogfu/skills/first-audit/`, or `dogfu/skills/berlin-theme/` (see each folder's `README.md`) — then reinstall/update the
`dogfu` plugin from the marketplace to pick up the changes. When shipping a change, bump
the `version` field in `dogfu/.claude-plugin/plugin.json`.
