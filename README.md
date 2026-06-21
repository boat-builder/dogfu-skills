# dogfu-skills

A Claude **Cowork plugin marketplace**. It currently ships one plugin:

| Plugin | What it does |
| :-- | :-- |
| **`dogfu`** | Sales toolkit powered by the `dogfu` CLI. Two skills: **lead-research** (**Discover → Qualify → Enrich → CRM** — turn a sparse sales target into a qualified, enriched Close CRM record) and **lead-touch** (work an existing lead through the outreach cadence in Close — pull what's due, record touches the BDR sent, mark replies, move statuses, report the funnel). |

## Layout

```
dogfu-skills/
├── .claude-plugin/
│   └── marketplace.json          # marketplace catalog (lists the plugins below)
└── dogfu/                         # the dogfu plugin (source directory)
    ├── .claude-plugin/
    │   └── plugin.json            # plugin manifest
    ├── .mcp.json                  # bundles the dogfu MCP server (auto-registers on install)
    └── skills/
        ├── lead-research/         # the lead-research skill (discover → qualify → enrich → CRM)
        │   ├── SKILL.md
        │   ├── README.md
        │   ├── references/
        │   └── icps/
        └── lead-touch/            # the lead-touch skill (CRM & outreach-cadence ops)
            ├── SKILL.md
            └── README.md
```

Skills are auto-discovered from each plugin's `skills/` directory — they are not listed
in `plugin.json`.

## Install in Cowork

In Cowork, open **Customize → Plugins**, add this repository as a marketplace, then
install the **Dogfu** plugin:

- **From a local path** — point the marketplace at this folder. No git remote required.
- **From git** — point it at this repo's URL once it's pushed somewhere reachable.

Once installed, the skills trigger automatically: **lead-research** when you give Claude a
sales target ("research this lead", "is this company in our ICP", a bare LinkedIn/company
link, etc.), and **lead-touch** when you want to work an existing lead in the CRM ("who do I
follow up with today", "I sent the LinkedIn DM to X", "mark this lead replied", "what's our
funnel look like").

## Requirement: the dogfu MCP + CLI

**The plugin bundles the dogfu MCP server** — a remote, OAuth-authenticated MCP at
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

Edit the markdown under each skill's folder — `dogfu/skills/lead-research/` or
`dogfu/skills/lead-touch/` (see each folder's `README.md`) — then reinstall/update the
`dogfu` plugin from this marketplace to pick up the changes. Bump `version` in
`dogfu/.claude-plugin/plugin.json` when you ship a change.
