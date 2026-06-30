# first-audit — skill source

This is the `first-audit` skill, shipped inside the **`dogfu`** plugin. It triggers when
someone explicitly asks for an **outside-in SEO/AEO audit of a prospect domain** (a domain
you do *not* own) as a sales deliverable: a public-site crawl, keyword/competitor/traffic
intelligence, and live Google/AI answers, assembled into **one visual single-page report**
published to a public bucket, with the audit URL recorded on the prospect's Close lead. It is
the **diagnosis only** — findings that make the gaps undeniable, ending in a single
book-a-call CTA — no "what to fix" section. It sits outside the lead-research → lead-touch →
lead-engage CRM lifecycle.

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
  bucket with its own UI** via a new `dogfu report publish` command, and the design guide is
  **vendored** into `references/design-guide.md` (verbatim copy of the backend's
  `internal/reports/design_guide.md`) instead of fetched live.
- **No project guardrails.** No "configure --domain", no "abort if the project doesn't exist."
  You audit the domain directly.

The entire data + crawl + AEO layer maps onto existing `dogfu` commands with no backend work —
see `references/data-sources.md`.

## Files
- `SKILL.md` — the operating manual (runtime only): pre-flight, the six-phase pipeline
  (footprint → brand brief + crawl → seeds → keyword/competitor → answer-engine → collect →
  build/publish/record), and the cost/latency guardrails. The YAML frontmatter
  `name`/`description` controls when the skill triggers (invoked explicitly).
- `references/data-sources.md` — the `dogfu` `seo` / `google` / `chatgpt` command map (flags,
  output fields, localization, what dogfu does NOT cover).
- `references/bluesnake.md` — the Bluesnake crawl lifecycle + SQLite query cookbook
  (technical + AEO on-page signals). Copied verbatim from the original — Bluesnake is the same
  MCP in both ecosystems.
- `references/report.md` — how the report is assembled and shipped: the main agent collects
  findings into `<run-dir>/findings/*.md` as it goes, a **report-builder sub-agent** renders
  one `index.html` from those markdowns + the design guide + the Step 3 page layout, then the
  main agent `dogfu report publish`es it and attaches the URL to Close. Step 3 is the existing
  layout recommendation (no separate layout artifact exists).
- `references/design-guide.md` — the authoritative, vendored design spec (colors, type,
  components, ECharts theme) the report is built against.

## Dependencies (beyond the dogfu CLI)
1. **Bluesnake MCP** — local crawler/auditor, **not bundled** (configure as a separate MCP,
   user/project scope), same as the original plugin. Supplies the crawl + on-page layer.
2. **`dogfu report publish`** — the public-bucket publish command, **being added to the CLI**.
   Contract: `dogfu report publish --domain <d> --html <file>` → JSON with a public `url`; the
   CLI verifies the book-a-call CTA (`https://cal.link/berlin`) is in the HTML and rejects the
   report otherwise. Reconcile flags against `dogfu report publish --help` once it ships. Until
   then the skill assembles the report but won't fabricate a URL.

CRM writes (just the audit URL, as a `crm note`) use the Close API key set once in the admin
**Console → CRM Integration**, like the other dogfu skills.

## How to edit
1. Edit `SKILL.md` (or the references) — or ask Claude to.
2. To change *when it fires*, edit the `description` frontmatter. To change *what it pulls*,
   edit `references/data-sources.md`. To change *how the report looks*, edit
   `references/design-guide.md` — but keep it in sync with the upstream
   `backend/internal/reports/design_guide.md` if that's still the source of truth for the
   bucket UI.
3. When `dogfu report publish` ships, reconcile `references/report.md` Step 5 with its real
   flags/output.

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update the
plugin from this marketplace (see the repo-root `README.md` for install steps).
