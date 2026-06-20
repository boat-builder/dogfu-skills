# lead-research — skill source

This is the `lead-research` skill, shipped inside the **`salesx`** plugin. It triggers
automatically when you hand Claude a sales target (a company, domain, person, or
LinkedIn/X URL) and ask to research, qualify, enrich, or CRM it.

## Files
- `SKILL.md` — the operating manual: the four stages, how to invoke `salesx`, the capability→command map, and the CRM write rules. The YAML frontmatter `name` / `description` controls when the skill triggers.
- `references/pipeline.md` — how to interpret the qualification (the B1–B8 phases, synthesis, calibration). Read before running Stage B.
- `references/salesx-commands.md` — the `salesx` command catalog (flags + output fields + market codes).

## The `salesx` CLI dependency
The skill is the orchestration layer; the data and CRM actions come from the `salesx`
CLI, which holds its own API credentials. The skill **does not bundle `salesx`** — it
shells out to it. `salesx` is a `uv`-managed CLI in a local directory that you mount into
your Cowork session; the skill runs it from inside that directory (`uv run salesx …`), and
credentials stay in the `.env` beside it. See the **Running `salesx`** section of
`SKILL.md` for the exact invocation.

## How to edit
1. Edit the markdown files in this folder (or ask Claude to).
2. Keep `SKILL.md` the orchestration layer; put deep detail in `references/`.
3. To change *what it does*, edit `SKILL.md` / `references/`. To change *when it fires*, edit the `description` in the `SKILL.md` frontmatter.

Changes take effect wherever the `salesx` plugin is installed once you reinstall/update
the plugin from this marketplace (see the repo-root `README.md` for install steps).
