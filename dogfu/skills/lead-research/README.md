# lead-research — skill source

This is the `lead-research` skill, shipped inside the **`dogfu`** plugin. It triggers
automatically when you hand Claude a sales target (a company, domain, person, or
LinkedIn/X URL) and ask to research, size up, enrich, or CRM it.

The flow is staged around a human decision: a cheap **Scout** pass gathers the signals,
a succinct **checkpoint brief** puts the **pursue-or-drop** call in the user's hands, and
only a *pursue* unlocks the expensive **deep dive** (competitive gap, AEO checks, top
pages, decision-maker reads, verified emails). The skill never sets a status or writes to
the CRM on its own: a lead lands in Close — Qualified on a pursue, Bad Fit on a drop, with
structured company attributes attached — only once the user has made that call.

## Files
- `SKILL.md` — the operating manual: the staged flow, the decision aids, how to invoke
  `dogfu`, the capability→command map, the checkpoint-brief format, and the CRM hand-off
  (the write itself runs through the crm skill's `references/intake.md`). The YAML
  frontmatter `name` / `description` controls when the skill triggers.
- `references/pipeline.md` — how to interpret each phase (Scout S1–S9, deep dive
  D1–D6), how to compose the brief, and the attribute→source map for the CRM fields.
  Read before running the Scout.
- `references/dogfu-commands.md` — the `dogfu` command catalog (flags + output fields
  + market codes).

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
