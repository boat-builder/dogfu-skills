# lead-research — skill source

This is the `lead-research` skill, shipped inside the **`dogfu`** plugin. It triggers
automatically when you hand Claude a sales target (a company, domain, person, or
LinkedIn/X URL) and ask to research, qualify, enrich, or CRM it.

## Files
- `SKILL.md` — the operating manual: the four stages, how to invoke `dogfu`, the capability→command map, and the CRM write rules. The YAML frontmatter `name` / `description` controls when the skill triggers.
- `references/pipeline.md` — how to interpret the qualification (the B1–B8 phases, synthesis, calibration). Read before running Stage B.
- `references/dogfu-commands.md` — the `dogfu` command catalog (flags + output fields + market codes).
- `icps/` — the Ideal Customer Profiles the skill qualifies against (one file per ICP, with a default). The skill loads one automatically, so you don't paste an ICP into every request. See `icps/README.md` for the convention.

## The `dogfu` CLI dependency
The skill is the orchestration layer; the data and CRM actions come from the `dogfu` CLI.
The skill **does not bundle `dogfu`** — it shells out to it. `dogfu` is a published package
(`pip install dogfu`) that you install and authenticate through the **dogfu MCP**: connect
the MCP, run its `get_setup_instructions` tool, install the CLI, then `dogfu configure --otp
<OTP> --title "…"` to save a session token. CRM (Close) calls go through the backend under
the Close API key you set once in the admin **Console → CRM Integration**. There is no
directory to mount and no `.env`. See the **Running `dogfu`** section of `SKILL.md` for the
exact flow.

## How to edit
1. Edit the markdown files in this folder (or ask Claude to).
2. Keep `SKILL.md` the orchestration layer; put deep detail in `references/`.
3. To change *what it does*, edit `SKILL.md` / `references/`. To change *when it fires*, edit the `description` in the `SKILL.md` frontmatter.

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update
the plugin from this marketplace (see the repo-root `README.md` for install steps).
