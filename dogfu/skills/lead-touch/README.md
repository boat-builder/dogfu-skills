# lead-touch — skill source

This is the `lead-touch` skill, shipped inside the **`dogfu`** plugin. It triggers when a
BDR wants to interact with the Close CRM about leads that already exist — check a lead's
status or cadence position, pull the leads due for a follow-up, record a touch they just
sent, mark a lead replied / not interested, change a status, read touch history, or manage
notes / tasks / contacts. It's the layer that runs **after** `lead-research`: research adds
and qualifies a lead; lead-touch works it through outreach.

## Files
- `SKILL.md` — the operating manual: how our leads are organized in Close (the funnel-status vs cadence-state model, the four system-managed touch fields, the cadence table, the "due" rule), how to run `dogfu`, the full `dogfu crm` command surface, and the operating rules. The YAML frontmatter `name` / `description` controls when the skill triggers.

There are no `references/` or `icps/` here — unlike `lead-research`, this skill is a single
self-contained file. The command surface is inline so the agent rarely needs `dogfu --help`.

## The `dogfu` CLI dependency
Same dependency as `lead-research`: the skill is the orchestration layer and shells out to
the `dogfu` CLI for all data and CRM actions. The skill **does not bundle `dogfu`** — it's a
published package (`pip install dogfu`) installed and authenticated through the **dogfu MCP**
(`get_setup_instructions` → `dogfu configure --otp <OTP> --title "…"`). CRM (Close) calls go
through the backend under the Close API key set once in the admin **Console → CRM
Integration**. See the **Running `dogfu`** section of `SKILL.md`.

## The cadence lives in the CLI, not here
The channel sequence and waits (LinkedIn connection → LinkedIn DM → email → X, with the
between-touch waits) are defined in the CLI's own `dogfu.cadence` config and applied
automatically by `dogfu crm touch record`. The table in `SKILL.md` documents the current
cadence so the agent can reason about it — but the CLI is the source of truth. If the cadence
config changes, update the table in `SKILL.md` to match.

## How to edit
1. Edit `SKILL.md` (or ask Claude to).
2. To change *what it does*, edit the body. To change *when it fires*, edit the `description`
   in the frontmatter.
3. Keep it tight and self-contained — the command semantics inline are what let the agent
   avoid `--help`.

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update the
plugin from this marketplace (see the repo-root `README.md` for install steps).
