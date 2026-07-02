# Building & publishing the single-page report

The deliverable is **one** self-contained HTML page, published to Berlin's public report
bucket via the `dogfu` CLI, which returns a public URL you then attach to the prospect's
Close CRM lead. Build it to Berlin's house design system.

**Who builds what.** The main agent gathers every finding into markdown files under
`<run-dir>/findings/` as the audit runs (see `SKILL.md`). Phase F then spawns a
**report-builder sub-agent** that turns those markdowns into the HTML against one design
reference: **`references/design-guide.md`** — the visual atoms (Step 1) plus the page
composition at the end of that file (Step 3). It builds from the markdown only — it doesn't
re-run any data call. The **main agent** then publishes (Step 5) and records the URL (Step 6).
Steps 1–4 are for whoever assembles the page (the sub-agent); Steps 5–6 are the main agent's.

## Step 1 — the design guide is `references/design-guide.md` (don't reinvent it)

Read **`references/design-guide.md`** and follow it *verbatim*. It is the authoritative
spec for every color, font, size, component, and chart style. (In the original agentberlin
flow this was fetched live with `agentberlin report design`; the dogfu publish path has no
live design endpoint, so the guide is vendored into this skill instead — same content.)

The guide covers both the **visual atoms** and, in its final *Composition* section, the
**cold-lead layout** — reading order, the two-tempo hierarchy, and the three attention
components that make the page skim in ~10 seconds (all in the same tokens).

Orientation, so you know what to expect (confirm every value against the guide):

* **Tokens** on `:root`: `--bg #f6f5f2` (cream canvas), `--surface #fff`, text `--ink`/`--ink-2/3/4`, signature `--accent #7c4dff`/`--accent-2 #5b2fd1`, state `--ok`/`--warn`/`--danger` (+ `-soft`).
* **Fonts:** serif *Instrument Serif* (titles + big numbers), sans *Inter* (body), mono *IBM Plex Mono* (labels/chips/timestamps).
* **The only external resources allowed** are the three Google Fonts and the two Berlin chart scripts (ECharts + the `berlin` theme from `files.agentberlin.ai`). No Chart.js, no other CDN, no images/icon libraries. These load fine from any host, so they work unchanged from the public bucket.
* **Components:** one hero card (the headline scorecard), plain + severity-tinted cards (crit/warn/info) for findings, stat tiles with thin progress bars for sub-scores, mono pills/chips for tags, list rows for tables, two-tone `<h1>headline <em>accent</em></h1>` headings. Layout `max-width:1120px`; one responsive breakpoint at 820px.
* **Tone:** keep the body analytical and third-person — it reads as credible. The one intentional promotional element is a single **book-a-call CTA** at the very end (Step 3); the report's job is to make the gap undeniable, then point to a conversation. (This is the deliberate exception to the design guide's "no CTAs" rule — nothing else on the page is promotional.)

## Step 2 — charts

* **Single-value indicators** (the hero score dial, a lone progress bar): hand-authored **inline SVG** using the radial-arc / thin-bar recipes in the design guide. No library.
* **Everything backed by a data series** (competitor traffic bars, status-code donut, CWV gauges, cited-source bars, an AEO presence heatmap): **Apache ECharts** with the registered `berlin` theme. Load once, tokens/echarts before theme:
  ```html
  <script src="https://files.agentberlin.ai/vendor/echarts/5.6.0/echarts.min.js"></script>
  <script src="https://files.agentberlin.ai/vendor/berlin-theme/v1/theme.js"></script>
  ```
  Init with the SVG renderer + theme, then set type + data only (the theme owns styling):
  ```js
  echarts.init(el, 'berlin', { renderer: 'svg' }).setOption({ /* type + data */ });
  ```
  Give each chart container an explicit height (e.g. 300px) or ECharts renders nothing. Use a `gauge` for a data-driven score; keep the inline-SVG dial only for the static hero score. Helpers: `BERLIN.accentGradient`, `BERLIN.fadeGradient`, `BERLIN.tokens`.

## Step 3 — the page layout

**The layout — band order, hierarchy, whitespace tempo, and the attention components — lives in
the *Composition* section of `references/design-guide.md`. Build top to bottom from it.** In
short: a loud, airy **skim**
(verdict hero → three shock-stats → one dark spotlight) that lands in ~10 seconds and leads
with the findings a prospect's own tools can't surface (AEO, machine-readability), then the
calmer **evidence** sections (search, authority, technical). Each band renders one `findings/`
file — render its numbers, don't go looking for raw JSON, and omit a band cleanly (noting it
back to the main agent) if its file is missing rather than inventing numbers. Pull the brand's
name, positioning, and competitors from **`brand-brief.md`** so the copy is specific.
**Diagnosis only — no recommendations / "what to fix" section.**

## Step 4 — the output is a single HTML file

The whole audit is **one self-contained HTML file** at `<run-dir>/report.html` — all CSS
inline; the only external refs are the fonts + the two chart scripts from Step 2. No folder
and no sidecar files: the raw numbers already live in the `findings/` markdowns and the
`dogfu -o` dumps, so the page itself is the only thing to publish.

## Step 5 — publish to the bucket (main agent)

Publishing is the deliverable — run it directly (the skill was invoked to produce and publish
the audit; no separate approval step):

```bash
dogfu report publish --domain <prospect-domain> --html <run-dir>/report.html
```

It uploads the single HTML file to the public bucket and returns JSON with a public **`url`**
— capture it for Step 6. The CLI **requires** the book-a-call CTA (`https://cal.link/berlin`)
to be present in the page and rejects the report otherwise, so make sure Step 3's CTA made it
in.

> ⚠️ **Pending CLI.** `dogfu report publish` is being added. If it isn't available yet, stop
> here, tell the user the report is assembled at `<run-dir>/report.html`, and don't fabricate
> a URL. Reconcile the exact flags against `dogfu report publish --help` once it ships.

## Step 6 — attach the audit URL to the Close CRM lead

Record the published audit so the BDR can send it. Put it on the prospect's lead:

1. **Find the lead.** If you resolved it in Phase A0 (the reference-prior step), **reuse that `lead_id`** — don't search again. Otherwise find it by domain: `dogfu crm lead list -q <domain>` (most prospects already exist from lead-research). If none matches, create a minimal one: `dogfu crm lead create -n "<company>" -u <domain>` (don't set a status here — this skill only attaches the audit).
2. **Attach the URL** as a note: `dogfu crm note create <lead_id> -t "First audit published: <url> — overall grade <X>/100 (AEO <a> · Technical <t> · Visibility <v> · Authority <au>)."` Notes are HTML-escaped on write, so use plain text. (If you'd rather have it on the lead headline, you can instead/also drop it in the lead `description` via `crm lead update <lead_id> -d "…"` — but a note keeps the history and is the default home.)

CRM writes go through the caller's own Close key (Console → CRM Integration). A `412` "no
Close CRM API key configured" means it isn't connected — surface that and skip Step 6
rather than retrying; the audit is still published and you can hand the user the URL.

After publishing, tell the user the audit is **live at `<url>`** and that the link is
recorded on the `<company>` lead in Close.
