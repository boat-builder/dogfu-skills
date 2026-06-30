# dogfu-skills

A plugin marketplace for both **Claude Cowork** and **Codex**. It currently ships one plugin:

| Plugin | What it does |
| :-- | :-- |
| **`dogfu`** | Sales toolkit powered by the `dogfu` CLI. Five skills: **lead-research** (**Discover → Qualify → Enrich → CRM** — turn a sparse sales target into a qualified, enriched Close CRM record), **lead-touch** (work an existing lead through the **cold** outreach cadence in Close — pull what's due, record touches the BDR sent, mark replies, move statuses, report the funnel), **lead-engage** (the **warm** counterpart — once a lead engages, run discovery, open/advance/close an **opportunity** (Discovery → Trial → Proposal → Won/Lost), set the next step, surface deals due or stalled, and forecast the pipeline), **crm-cleanup** (read-only health audit — find the leads and tasks that fell out of the flow, each with the exact fix command), and **berlin-theme** (Berlin's brand style guide — apply the editorial cream/paper house style to any sales collateral you design: pages, one-pagers, posters, emails. No CLI; theme only). |

The three CRM-operating skills form one lifecycle: **lead-research** produces a qualified lead →
**lead-touch** runs cold outreach and hands off when the lead replies (→ *Engaged*) →
**lead-engage** works the live deal (a Close opportunity) to a close. **crm-cleanup** audits all
of it read-only. `lead-engage` additionally depends on opportunity support in the `dogfu` CLI +
a Close pipeline (see its `references/opportunity-cli-spec.md`).

## Layout

```
dogfu-skills/
├── .agents/
│   └── plugins/
│       └── marketplace.json      # Codex marketplace catalog
├── .claude-plugin/
│   └── marketplace.json          # Claude marketplace catalog
└── dogfu/                         # the dogfu plugin (source directory)
    ├── .claude-plugin/
    │   └── plugin.json            # Claude plugin manifest
    ├── .codex-plugin/
    │   └── plugin.json            # Codex plugin manifest
    ├── .mcp.json                  # shared dogfu MCP server configuration
    └── skills/
        ├── lead-research/         # the lead-research skill (discover → qualify → enrich → CRM)
        │   ├── SKILL.md
        │   ├── README.md
        │   ├── references/
        │   └── icps/
        ├── lead-touch/            # the lead-touch skill (cold CRM & outreach-cadence ops)
        │   ├── SKILL.md
        │   └── README.md
        ├── lead-engage/           # the lead-engage skill (warm phase — live deals / opportunities)
        │   ├── SKILL.md
        │   ├── README.md
        │   └── references/        # opportunity CLI spec + Close setup
        ├── crm-cleanup/           # the crm-cleanup skill (read-only CRM health audit)
        │   ├── SKILL.md
        │   └── README.md
        └── berlin-theme/          # the berlin-theme skill (brand style guide, no CLI)
            ├── SKILL.md
            └── README.md
```

Skills are auto-discovered from each plugin's `skills/` directory — they are not listed
in the Claude manifest. The Codex manifest points at the same shared `skills/` directory.

## Install in Cowork

In Cowork, open **Customize → Plugins**, add this repository as a marketplace, then
install the **Dogfu** plugin:

- **From a local path** — point the marketplace at this folder. No git remote required.
- **From git** — point it at this repo's URL once it's pushed somewhere reachable.

Once installed, the skills trigger automatically: **lead-research** when you give Claude a
sales target ("research this lead", "is this company in our ICP", a bare LinkedIn/company
link, etc.), **lead-touch** when you want to work an existing lead through *cold* outreach
("who do I follow up with today", "I sent the LinkedIn DM to X", "mark this lead replied",
"what's our funnel look like"), **lead-engage** once a lead has *engaged* ("a lead replied and
we had an intro call", "open a deal for X", "move this deal to trial", "which deals need action
today", "what's our pipeline / forecast", "an inbound lead reached out"), **crm-cleanup** for a
read-only health pass ("audit the CRM", "what's broken in Close", "find leads stuck in the
flow"), and **berlin-theme** when you ask Claude to design a Berlin-branded asset ("make this
landing page on-brand", "build a sales one-pager", "design a social poster in our brand").

## Install in Codex

Add this repository as a Codex marketplace, then install the `dogfu` plugin:

```bash
codex plugin marketplace add boat-builder/dogfu-skills
codex plugin add dogfu@dogfu-skills
```

Alternatively, after adding the marketplace, open `/plugins` in the Codex CLI or
**Plugins** in the Codex app and install **Dogfu** from **Dogfu Skills**. Start a new
thread after installation so Codex loads the bundled skills and MCP server.

For local development, replace the GitHub shorthand with the repository path:

```bash
codex plugin marketplace add /absolute/path/to/dogfu-skills
codex plugin add dogfu@dogfu-skills
```

The skills have the same triggers and behavior in Codex. You can also invoke the plugin or
one of its skills explicitly from the composer.

## Requirement: the dogfu MCP + CLI

**The plugin bundles the dogfu MCP server** for both Claude and Codex — a remote,
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

## Editing

Edit the markdown under each skill's folder — `dogfu/skills/lead-research/`,
`dogfu/skills/lead-touch/`, `dogfu/skills/lead-engage/`, `dogfu/skills/crm-cleanup/`, or `dogfu/skills/berlin-theme/` (see each folder's `README.md`) — then reinstall/update the
`dogfu` plugin from the relevant marketplace to pick up the changes. When shipping a
change, keep the `version` fields in `dogfu/.claude-plugin/plugin.json` and
`dogfu/.codex-plugin/plugin.json` in sync.
