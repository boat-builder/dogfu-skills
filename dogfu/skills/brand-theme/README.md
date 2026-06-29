# brand-theme — skill source

This is the `brand-theme` skill, shipped inside the **`dogfu`** plugin. It triggers when
someone asks you to design or build an **on-brand visual asset** — a landing or pitch
page, a sales one-pager, a social poster, ad creative, an email layout, a deck slide, a PDF,
or any HTML/CSS/graphic that should look and read on-brand. Unlike `lead-research` and
`lead-touch`, it does **not** shell out to the `dogfu` CLI — it carries no data and runs no
commands. Its only job is to **tell the agent the brand theme** so whatever the user asks for
comes out on-brand. The user owns the output format and content; this skill owns the look,
type, motion, and copy voice.

## Files
- `SKILL.md` — the brand brief: palette, typography, layout, components, motion, copy
  voice/tone, a quick-reference checklist, and drop-in CSS variables. The YAML frontmatter
  `name` / `description` controls when the skill triggers. This is the entire skill — a single
  self-contained file, no `references/` or `icps/`.

## Self-contained
`SKILL.md` is the complete, self-contained brand brief — it carries every token, rule, and
guideline an agent needs, with no pointers to external files or repos (agents using this skill
won't have the web app on hand). Keep it that way: when the brand evolves, edit the
values directly in `SKILL.md`.

## How to edit
1. Edit `SKILL.md` (or ask Claude to).
2. To change *what brand rules it teaches*, edit the body. To change *when it fires*, edit the
   `description` in the frontmatter.
3. Keep it tight and self-contained — the inline tokens and rules are what let the agent
   produce on-brand work without hunting for the CSS.

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update the
plugin from this marketplace (see the repo-root `README.md` for install steps).
