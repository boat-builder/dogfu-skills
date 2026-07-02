---
name: first-audit
description: >-
  Run a first-touch SEO/AEO audit of a prospect domain from public data and publish it as a
  single-page Berlin report — using the `dogfu` CLI and the Bluesnake MCP. Use this when
  someone wants an outside-in audit of a prospect (a domain you do NOT own) as a sales
  deliverable: "run a first audit on <domain>", "do an SEO/AEO audit for this prospect",
  "build the audit report for <company>", "audit <domain> and publish it". It crawls the
  public site, pulls keyword/competitor/traffic intelligence and live Google/AI answers,
  then publishes one visual single-page report and records the audit URL on the lead in
  Close. Invoked explicitly. For researching/qualifying a brand-new target use lead-research;
  for outreach cadence use lead-touch.
---

# First Audit — prospect SEO/AEO audit from public data

This skill produces a **first-touch audit of a domain you do not own** — a sales
deliverable for a potential customer. You build it from a live crawl of the prospect's site,
third-party keyword/traffic intelligence, and live Google / AI answers.

The output is **one single-page, visual report published to Berlin's public report bucket**
(cards, charts, tables — no walls of text), styled with Berlin's house design system, with
its public URL recorded on the prospect's Close lead. The report is the **diagnosis only**:
it makes the gaps undeniable and ends with a single call to action to book a call. It does
not include a recommendations / "what to fix" section — that conversation happens on the call.

## What you need before starting

* **The Bluesnake MCP** must be available.
* **Authenticate the `dogfu` CLI first.** Before any `dogfu` command, call the dogfu MCP's
  **`get_setup_instructions`** tool and follow it: install the `dogfu` package (if needed),
  then run `dogfu configure --otp <OTP> --title "First audit: <prospect-domain>"` with the OTP
  it returns (saved to `~/.dogfu/config.json`). There is **no project to register and no
  `--domain` to bind** — you audit the domain directly.
* The `dogfu` CLI is a stateless HTTP client. Output is **canonical JSON by default**; add
  `-o FILE` to dump large payloads to a file (you'll do this constantly). Discover flags
  with `dogfu <group> <cmd> --help`.

Publishing the report and recording its URL on the lead are the **expected deliverables** of
this skill — it is invoked explicitly to produce and publish a prospect audit, so just run
them. No separate write-approval step is needed.

## How the pipeline flows

The ordering matters because later steps depend on earlier ones, and two long-running things
(the crawl and the brand research) should run **in the background while you do other work**.
Read the reference files as you reach the phases that need them:

* `references/data-sources.md` — every `dogfu` command you'll call (`seo`, `google`, `chatgpt`), with flags, output fields, the localization policy, and cost/latency caveats.
* `references/bluesnake.md` — the crawl lifecycle and the SQLite query cookbook (technical + AEO signals).
* `references/report.md` — how to assemble the single-page report, publish it with `dogfu report publish`, and attach the URL to Close (incl. the book-a-call CTA).
* `references/design-guide.md` — the authoritative, vendored design spec the report is built against (colors, type, components, charts).

### Collect findings as you go — you write the markdown, a sub-agent writes the HTML

This audit gathers many separate information sets over a long run. **You (the main agent) do
all the gathering and analysis**, and as you finish each phase you distill that set into its
own short **markdown file** under `<run-dir>/findings/` — the *report-ready* numbers, tables,
and one-line verdicts, not raw JSON. Keep the raw `dogfu -o` dumps too (provenance), but the
findings markdowns are what the report is built from.

At the very end (Phase F) you hand this `findings/` set — plus `references/design-guide.md`
and the page layout in `references/report.md` (Step 3) — to a **report-builder sub-agent**
that renders the single HTML page. This keeps the heavy design-guide + HTML-assembly work out
of your context, and means the page is built from facts you've already settled.

Recommended files (one information set each; consolidate sensibly):

| File | Filled in | Holds |
| :-- | :-- | :-- |
| `brand-brief.md` | A2 | brand name, positioning/ICP, industry, model, size, geos, competitors, topics |
| `findings/scorecard.md` | rolling | overall grade + sub-scores (Visibility, AEO, Technical, Authority) and the numbers behind each |
| `findings/answer-engine.md` | D | AEO presence matrix, share-of-voice, most-cited third-party sources — **plus** the LLM-Mentions index-wide numbers *only if* Phase D's real-vs-synthetic check kept them |
| `findings/search-competitive.md` | A1, C | footprint, prospect-vs-competitor traffic, top keywords, opportunity gaps |
| `findings/authority.md` | C | backlink authority — referring domains, backlink rank, spam score, prospect vs competitors (feeds the Authority sub-score) |
| `findings/technical.md` | E | status codes, indexability, thin content, Core Web Vitals (mobile + desktop), top issues |
| `findings/onpage-aeo.md` | A1, E | schema/JSON-LD coverage + errors, missing/dup H1 & meta, JS-render parity, llms.txt |

Each file should be self-sufficient: the sub-agent should be able to build that report section
from the file alone, without re-reading any JSON.

### Phase A — Pre-flight, then start the two slow things

**A0. Pull the prospect's existing Close profile — a reference *prior*.** The prospect was
almost certainly run through `lead-research` first, which wrote a cheap SEO/AEO profile to
Close. Read it (read-only) and distil it into `<run-dir>/prior.md`:

* `dogfu crm lead list -q <domain>` → the `lead_id` (keep it — Phase F Step 6 reuses it). Pick
  the match whose `url` is the prospect domain; if none matches, skip A0 (cold domain — fine).
* `dogfu crm lead get <lead_id>` → curated fields `industry, employees, revenue,
  business_model, seo_pages` + `description, status_label`.
* `dogfu crm note list <lead_id>` → the write-up: SEO/AEO metrics, ranked keywords,
  competitors + competitive gap, AEO checks, verdict.

**Reference, not replacement.** Every number in the report comes from *this* run's fresh pulls
— never copy a CRM figure in. Use the prior only to **seed** the brand brief (A2), seed
keywords (Phase B), and competitor shortlist (Phase C), and to **cross-check** fresh findings.
If Close isn't connected (`412`) or no lead exists, skip A0 and audit cold — it's never a gate.

**A1. Estimate the footprint** so you know how long the crawl will take and can set
expectations. Run `dogfu google search --query "site:<domain>"` and read `total_results`
(the "About N results" estimate; nullable). Cross-check by web-fetching `<domain>/sitemap.xml`
(count `<loc>`; if it returns gzipped/binary, fall back to the `Sitemap:` line in
`<domain>/robots.txt`). Also web-fetch `<domain>/llms.txt` and record whether it exists — an
AEO signal for the report. Treat the page count as a range (cross-check vs the prior's
`seo_pages`, A0).

**A2. Research the brand, then write `brand-brief.md`.** Spawn a **sub-agent** to research
the prospect from public sources (their site, about/pricing pages, positioning, reviews) and
return structured facts. This keeps the heavy reading out of your main context. Give it
`prior.md` as a starting point — it must still verify against live sources, not just echo it.
Then write the result to **`<run-dir>/brand-brief.md`** — a local markdown file, **not** a CRM
or platform write.
Capture, at minimum:

* **name** + **positioning / value prop / ICP** (the richest part — what they do, who they serve, the problem they solve)
* **industry** and **business model** (e.g. SaaS, marketplace, agency, non-profit)
* **company size** and **target customer segments**
* **geographies** they serve (drives every geo-aware call — see localization)
* **competitors** (the shortlist you'll validate in Phase C)
* **core content topics**

This brief **gates the rest** — write it before phases B–D. It makes every later step smart:
it seeds keyword choices, competitor selection, and answer-engine queries, and it supplies
the brand context the report copy is written from. (Nothing validates it server-side — no
fixed enums to satisfy; just capture accurate, useful facts.)

**A3. Start the Bluesnake crawl of the prospect** and let it run in the background. It returns
a `crawl_id` immediately; you'll collect results in Phase E. Create the Bluesnake project
first so competitors can be attached later. Only one crawl runs at a time, so kick the
prospect's off first. See `references/bluesnake.md`.

### Phase B — Seed keywords

From what you now understand about the brand (`brand-brief.md`), choose **up to 5 high-intent
keywords** — the terms a ready-to-act customer would search. For a commercial brand these are
commercial/transactional; for a content/non-profit brand they're the highest-intent discovery
terms. Your judgment, informed by the brand brief (the prior's ranked keywords in `prior.md`
are a starting list — validate and extend, don't just adopt). These seeds drive both the
keyword analysis and the answer-engine tests.

### Phase C — Keyword & competitor intelligence (`dogfu seo`)

Now start the paid data (see `references/data-sources.md`):

* **Pick ≤3 competitors.** There's no competitor-discovery endpoint, so shortlist from the brand research **and the prior's competitors** (`prior.md`), weighing **geography, industry, and ICP**, then validate each: do they rank for an overlapping set of the prospect's keywords, are they in a comparable traffic band, and do they show up in the same answer-engine results (Phase D)? Drop ones that pass none.
* Pull `seo ranked-keywords` for the prospect **and** each competitor, `seo keyword-ideas` from your seeds (opportunity gaps), `seo domain-overview` + `seo bulk-traffic-estimation` for the head-to-head, and optionally `seo historical-rank-overview` (momentum) and `seo technologies` (stack).
* **Authority (cheap — always run):** `seo backlinks-summary` for the prospect **and each competitor** (~$0.02 each) plus `seo referring-domains` for the prospect → real numbers for the **Authority** scorecard dimension (referring domains, backlink rank, spam score) instead of the ambiguous `technologies.domain_rank`. Write them to `findings/authority.md`.
* Attach the competitors to the Bluesnake project and start their crawls (sequential, after the prospect's).

Write every raw response to a file (`dogfu … -o <run-dir>/<name>.json`) and work from the
file — these are large and metered.

### Phase D — Answer-engine visibility (the AEO half)

This is what makes the audit current: is the brand showing up where AI answers are formed?
(Commands in `references/data-sources.md`.)

* Run your **seed keywords** through `dogfu google search` (read organic `results` + the inline `features.ai_overview`, `features.answer_box`, `features.related_questions`).
* Build **conversational variants — 2 per seed, up to 10** (how a person phrases a question to an assistant), and ask each in two places: **ChatGPT** via `dogfu chatgpt search --model gpt-5.4-mini` and **Google AI Mode** via `dogfu google ai-mode`.
* For every answer capture three things: did the **brand** appear? did a **competitor** appear? and which **domains were cited**. Aggregate the cited domains to find the **top third-party sources** (Reddit, YouTube, review sites…) — where the prospect would need to earn citations.

**Then cross-check against the LLM-Mentions index — and decide, per brand, whether it earns a place in the report.** The mentions API (`seo mentions-summary` / `mentions-top-domains` / `mentions-top-pages`, see `data-sources.md`) gives a deterministic view aggregated across *many* keywords — but it's keyword-indexed (ChatGPT US + Google AI Overview only), thin for niche/non-US brands, and costs ~5–10× a live call. So its raw output is a lead, not a verdict:

* Pull `seo mentions-summary --domain` for the **prospect and each competitor**, and `seo mentions-top-domains` / `mentions-top-pages` on your seeds. Dump each to `-o` files.
* **Run a dedicated real-vs-synthetic evaluation** — hand a **sub-agent** *both* this phase's live synthetic-query results **and** the mentions pulls (keeping the comparison out of your main context), and ask one question: *does the index data agree with, and add signal beyond, what the live queries already show for THIS brand?* It returns a verdict:
  * **Useful** — the brand or its category is genuinely indexed, the numbers are non-trivial and broadly consistent with the live reads → fold the index-wide **share-of-voice** (prospect vs competitors) and its **top-cited domains** into `findings/answer-engine.md`, clearly labeled as *index-wide* data (distinct from the live-query matrix).
  * **Too thin / misleading** — near-zero only because the brand isn't tracked, or it contradicts the live reads with no defensible reason → **omit it from the report.** Record one line in `findings/answer-engine.md` that it was checked and dropped, and why. A hollow "0 AI mentions" that just means "not indexed" would misrepresent the prospect — never ship it.

The live synthetic queries remain the report's **primary** AEO evidence (the actual answer a prospect sees today); the mentions index rides along only when it genuinely strengthens the picture.

### Optional — Brand narrative coherence (only on explicit request)

**Do not run by default.** *Only if the user explicitly asks* for a brand narrative /
category-coherence check, spawn a **sub-agent** to execute
`references/brand-narrative-coherence.md`, handing it `brand-brief.md`, the geo decision, the
competitor shortlist, Phase D's AI-answer citations, and output `findings/narrative-coherence.md`.
The sub-agent does all the work and writes that file; fold it in as an extra report section in
Phase F. You don't read the playbook yourself. Without a clear
ask, skip this entirely.

### Phase E — Collect the crawl(s) + technical signals

The prospect crawl is likely done; competitor crawls may still be running. **Wait for them to
finish unless the user says not to.** Then (see `references/bluesnake.md`):

* `issue_summary` for the audit verdict, and `query` (read-only SQL) for the specifics — status codes, indexability, thin content, internal linking, **and the AEO on-page signals**: schema/JSON-LD coverage + validation errors, missing/duplicate H1 & meta, JS-render parity, llms.txt.
* `project_comparison` for the prospect-vs-competitor technical scorecard.
* `dogfu seo lighthouse` on the key URL(s), **mobile and desktop** (separate `--strategy` calls), for Core Web Vitals + opportunities.

### Phase F — Build (sub-agent), publish & record the single-page report

Follow `references/report.md`. The build itself is delegated, so the design-guide + HTML work
stays out of your context:

1. **Spawn a report-builder sub-agent.** Hand it: the `<run-dir>/findings/` markdowns + `brand-brief.md` (the content), `references/design-guide.md` (how it looks), and the page layout in `references/report.md` Step 3 (what sections go where — our existing layout recommendation). Its job: assemble **one self-contained HTML file** at `<run-dir>/report.html` — hero scorecard + the findings sections, ECharts for data-series charts, inline SVG for single-value dials, ending with the book-a-call CTA → <https://cal.link/berlin> (required — publish rejects a report without it). **Findings only, no recommendations section.** It builds from the markdown you already wrote — it should not need to re-run any `dogfu` call. Have it return the path to the HTML and flag anything a findings file left ambiguous.
2. **Publish** (you, the main agent): **`dogfu report publish --domain <prospect-domain> --html <run-dir>/report.html`** → returns the public URL. Capture it.
3. **Record** (you, the main agent): **attach that URL to the prospect's Close lead** (`dogfu crm note create`).

## Guardrails that keep this fast and cheap

* **`dogfu` calls can take minutes** (especially the first after an idle period). Budget a generous timeout and don't assume a hang means failure. (Warm calls are usually a few seconds.)
* **Never fetch the same thing twice.** Write raw outputs to JSON files immediately with `dogfu -o`, then read/aggregate from disk — protects both your context and the API budget.
* **Quotas & cost:** `seo lighthouse` blocks ~10–90s; the `seo` data calls are metered — cap `--limit`, use `--filters`/`--order-by`. Don't pull data the report won't use.
* **Backlinks are cheap; Mentions are not.** The `seo backlinks-*` calls run ~$0.02 — pull them freely for the Authority score. The `seo mentions-*` calls run ~$0.10–0.25 **and** are keyword-indexed (ChatGPT US + Google AI Overview only), so they're gated on the Phase D real-vs-synthetic check: keep them in the report only when they add real signal for that brand, otherwise drop them (and note the drop). Never let a thin index return produce a misleading "0 mentions" claim.
* **One Bluesnake crawl at a time** — prospect + up to 3 competitors run sequentially, so start early and work on phases B–D while they run.
* **Localization:** match the brand's geography (or go global if multi-region) consistently across every geo-aware call (`seo` `--location-code`/`--language-code`, `google` `--country`/`--language`/`--location`, `chatgpt` `--user-country`) — see `references/data-sources.md`.
* **Keep context lean:** use sub-agents for brand research and any other heavy reading (return only distilled facts), and offload the final report-HTML assembly to the report-builder sub-agent (Phase F) — you gather and decide into the `findings/` markdowns, it renders.
* **One run dir.** Make a single directory with `mktemp -d` at the start and put *everything* this run writes inside it — `dogfu -o` dumps, `brand-brief.md`, the `findings/` markdowns, and `report.html`. Never a fixed or target-derived path (parallel runs would clobber it).
