# first-audit — skill source

This is the `first-audit` skill, shipped inside the **`dogfu`** plugin. It triggers when
someone explicitly asks for an **outside-in SEO/AEO audit of a prospect domain** (a domain
you do *not* own) as a sales deliverable: a public-site crawl, keyword/competitor/traffic
intelligence, and live Google/AI answers, assembled into **one visual single-page report**
published to a public bucket, returning its public URL. It is the **diagnosis only** —
findings that make the gaps undeniable, ending in a single book-a-call CTA — no "what to
fix" section. The skill audits and publishes, nothing else.

## Ported from the Agent Berlin plugin

This skill was ported from the standalone `agent-berlin` plugin (the `first-audit` skill that
ran on the `agentberlin` CLI/SDK + MCP). Both ecosystems hit the **same** Agent Berlin
backend; the port swaps the agentberlin SDK/CLI for the `dogfu` CLI and drops the two
platform-coupled steps:

- **Brand profile.** The original wrote a brand profile to the prospect's *Berlin project*
  (`client.brand.update_profile`). dogfu has no project concept, so we **don't** write
  anywhere — the brand research still happens (a sub-agent reads public sources) but is
  persisted to a local **`brand-brief.md`** in the run dir and used exactly as before (it
  seeds keyword/competitor/AEO choices and the report copy).
- **Report publishing.** The original published into Berlin's internal report platform, with
  the design fetched live. dogfu publishes a single self-contained HTML file to a **public
  bucket with its own UI** via a new `dogfu report publish` command, and the design guide lives
  in `references/design-guide.md` instead of being fetched live. It **began** as a vendored copy
  of the backend's `internal/reports/design_guide.md`, but is now **owned by this skill and
  evolves on its own** — no upstream sync — and has since grown a cold-lead *Composition*
  section the backend guide never had.
- **No project guardrails.** No "configure --domain", no "abort if the project doesn't exist."
  You audit the domain directly.

The entire data + crawl + AEO layer maps onto existing `dogfu` commands with no backend work —
see `references/data-sources.md`.

## Files
- `SKILL.md` — the operating manual (runtime only): pre-flight, the six-phase pipeline
  (gather any prior research → footprint → brand brief + crawl → seeds → keyword/competitor →
  answer-engine → collect → build/publish), and the cost/latency guardrails. Phase A0
  distils any prior research already at hand (e.g. a lead-research run) into a **reference
  prior** — a warm start that seeds the brand brief, seed keywords, and competitor shortlist,
  never a substitute for this run's fresh audit data. The YAML frontmatter `name`/`description`
  controls when the skill triggers (invoked explicitly).
- `references/data-sources.md` — the `dogfu` `seo` / `google` / `chatgpt`
  command map (flags, output fields, localization, what dogfu does NOT cover).
- `references/bluesnake.md` — the Bluesnake crawl lifecycle + SQLite query cookbook
  (technical + AEO on-page signals). Copied verbatim from the original — Bluesnake is the same
  MCP in both ecosystems.
- `references/report.md` — how the report is assembled and shipped: the main agent collects
  findings into `<run-dir>/findings/*.md` as it goes, a **report-builder sub-agent** renders
  one `index.html` from those markdowns + the design guide (atoms + its *Composition* section),
  then the main agent `dogfu report publish`es it and hands the user the URL.
- `references/design-guide.md` — the authoritative design spec the report is built against:
  the visual atoms (colors, type, components, ECharts theme) **and**, in its final *Composition*
  section, the cold-lead layout — hook-first reading order, the two-tempo (skim → evidence)
  hierarchy, and three attention components (verdict hero, shock-stat strip, dark spotlight)
  that make the report skim in ~10s and lead with the AEO / machine-readability findings — the
  edge *felt*, not badged. The composition adds no new tokens.

## Dependencies (beyond the dogfu CLI)
1. **Bluesnake MCP** — local crawler/auditor, **not bundled** (configure as a separate MCP,
   user/project scope), same as the original plugin. Supplies the crawl + on-page layer.
2. **`dogfu report publish`** — the public-bucket publish command, **being added to the CLI**.
   Contract: `dogfu report publish --domain <d> --html <file>` → JSON with a public `url`; the
   CLI verifies the book-a-call CTA (`https://cal.link/berlin`) is in the HTML and rejects the
   report otherwise. Reconcile flags against `dogfu report publish --help` once it ships. Until
   then the skill assembles the report but won't fabricate a URL.

## How to edit
1. Edit `SKILL.md` (or the references) — or ask Claude to.
2. To change *when it fires*, edit the `description` frontmatter. To change *what it pulls*,
   edit `references/data-sources.md`. To change how the report **looks or is laid out** — the
   visual atoms (colors, type, components, charts) or the cold-lead *Composition* (reading
   order, hierarchy, the skim/attention components) — edit `references/design-guide.md`. It's
   this skill's own file now (no upstream to keep in sync with), so it can evolve freely.
3. When `dogfu report publish` ships, reconcile `references/report.md` Step 5 with its real
   flags/output.

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update the
plugin from this marketplace (see the repo-root `README.md` for install steps).
