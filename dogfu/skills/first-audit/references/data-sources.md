# Data sources (`dogfu` CLI)

Every call here is a **read-only** lookup against the Agent Berlin backend via the `dogfu`
CLI. Run any command from anywhere on your PATH once the dogfu MCP's `get_setup_instructions`
flow has installed and configured the CLI:

```bash
dogfu <group> <cmd> [flags]
```

JSON to stdout by default. Discover exact flags with `dogfu <group> <cmd> --help`.

**Golden rule:** these responses are large and metered. In every step, fetch once with
`dogfu … -o <run-dir>/<name>.json`, print a short summary, and do all later analysis from
the file. Budget a generous timeout per call (the first call after an idle period can take
a minute or two; warm calls are a few seconds). Never fetch the same thing twice.

The groups this skill uses: **`seo`** (keyword/competitor/traffic/technical), **`google`**
(Google web + AI Mode), **`chatgpt`** (the ChatGPT answer surface), and **`crm`** (Close —
to **read the lead-research profile as a reference prior at the start** and attach the
published audit URL at the end). The crawl/on-page layer is the **Bluesnake MCP**, documented
separately in `bluesnake.md`.

## Contents

* [crm — read the lead-research prior (Phase A0)](#crm)
* [seo — keywords, competitors, traffic, tech, CWV](#seo)
* [google — Google + AI surfaces](#google)
* [chatgpt — ChatGPT answers with citations](#chatgpt)
* [Localization policy](#localization)
* [What `dogfu` does NOT cover](#not-covered)

***

## crm

Read-only, used **once at the start** (Phase A0) to pull the SEO/AEO profile lead-research
already wrote to Close, as a **reference prior** — never a substitute for this run's fresh
data. All go through the caller's Close key (Console → CRM Integration); a `412` "no Close CRM
API key configured" means Close isn't connected — skip the prior and audit cold.

* **`crm lead list -q <domain>`** `[-n name] [-s status_id] [-l N] [--sort ..]` — resolve the
  prospect's lead by domain (no separate `search` verb — `list` with `-q` is the search).
  Take the `lead_id`; reuse it for the Step 6 attach so you don't look it up twice.
* **`crm lead get <lead_id>`** → the lead incl. the **curated company fields** lead-research
  set: `industry, employees, revenue, business_model, seo_pages`, plus `name, url,
  description, status_label, contacts[]`.
* **`crm note list <lead_id>`** `[-l N]` → the lead-research write-up notes — the SEO/AEO
  metrics, ranked/top keywords, chosen competitors + competitive gap, AI-answer visibility
  checks, and the fit verdict.
* **`crm contact list <lead_id>`** *(optional)* → the decision-makers and their links (context
  only; the report itself is domain-level).

The audit records its own output back via **`crm note create`** at the end — see `report.md`
Step 6.

***

## seo

Works for any domain — no ownership needed. Cap `--limit`, use `--order-by` / `--filters`.
Market is set with `--location-code` (numeric) and `--language-code` (two-letter) — see
[Localization](#localization).

* **`seo ranked-keywords --target <domain>`** `[--location-code N] [--language-code xx] [--limit 100] [--offset] [--order-by "keyword_data.keyword_info.search_volume,desc"] [--filters JSON]` — keywords a domain ranks for: `keyword, search_volume, cpc, competition, competition_level, rank, rank_group, url, title, intent, etv`. Run for the **prospect and each competitor**.
* **`seo keyword-ideas --keyword "<seed>"...`** `[--location-code] [--language-code] [--limit] [--order-by]` — expand your Phase B seeds into related ideas: `keyword, search_volume, cpc, competition, competition_level, difficulty` → opportunity gaps.
* **`seo domain-overview --target <domain>`** `[--location-code] [--language-code]` — organic snapshot: `organic{count (≈ ranking-keyword count), etv (traffic value), paid_traffic_cost, is_new/up/down/lost}, paid{}`.
* **`seo bulk-traffic-estimation --target <a>... --target <b>...`** `[--location-code] [--language-code] [--item-type organic ..]` — estimate traffic for up to 1,000 domains in one call → the head-to-head benchmark. Per target: `organic{count, etv}, paid`.
* **`seo relevant-pages --target <domain>`** `[--location-code] [--language-code] [--limit] [--order-by "metrics.organic.etv,desc"]` — the domain's top pages by organic traffic: `page_address, organic{count, etv, pos_*}, paid`.
* **`seo historical-rank-overview --target <domain>`** `[--location-code] [--language-code] [--date-from YYYY-MM-DD] [--date-to ..]` — monthly time series (momentum): `RankHistoryPoint[]` with `year, month, organic{count, etv, pos_1, pos_2_3, pos_4_10, pos_11_plus, ...}, paid`.
* **`seo technologies --target <domain>`** — CMS / analytics / framework stack + contact signals: `domain_rank` (**a large DataForSEO-style integer, NOT a 0–100 score — use only as a coarse, relative signal** between target and competitors), `last_visited, country_iso_code, emails[], phone_numbers[], social_graph_urls[], technologies{category:{subcategory:[...]}}`.
* **`seo lighthouse --url <url>`** `[--strategy mobile|desktop] [--locale en-US]` — Lighthouse / PageSpeed audit (this is the Core Web Vitals source): `performance_score, accessibility_score, seo_score, best_practices_score` (0–1, ×100 to show), `core_web_vitals{lcp_ms, fcp_ms, cls, tbt_ms, ...}`, real-user `field_data` (CrUX — **the signal Google actually ranks on; absent/None for low-traffic sites**, fall back to lab `core_web_vitals`), `opportunities[]`, `diagnostics[]`. Run **mobile and desktop separately** (two calls) on the key URL(s) — usually the homepage and a top landing page. Each call blocks ~10–90s.

***

## google

SERP feature fields default to empty/None when Google doesn't return them, so guard with
`if features.get("ai_overview"):` etc.

* **`google search --query "<q>"`** `[--country us] [--language en] [--location "City,Region,Country"] [--max-results 1-100]` → `query, total_results` (Google's "About N results" estimate — what `site:` queries read; nullable), `results[]` (title, url, snippet, displayed_link, position), and a **`features{}`** dict carrying `answer_box`, `related_questions` (People Also Ask), `knowledge_graph`, `ai_overview` (the inline AI Overview block when Google returns one), `related_searches`, `local_results`, etc.
* **`google ai-mode --query "<q>"`** `[--country] [--language] [--location]` → `query, answer` (markdown — the primary signal) + `citations[]` (title, url, source, snippet). This is Google's dedicated AI Mode.

**There is no separate `ai-overview` command** — the inline AI Overview rides in the
`features.ai_overview` field of `google search`. For the conversational AI surface use
`google ai-mode`.

**AEO use:** run seeds through `google search` (read `features.ai_overview` /
`features.answer_box` / `features.related_questions`); run conversational variants through
`google ai-mode`. For each, check brand & competitor presence and collect cited domains.

***

## chatgpt

* **`chatgpt search --query "<q>"`** `[--model gpt-5.4-mini] [--search-context-size low|medium|high] [--domain-filter <d>...] [--reasoning-effort ..] [--city/--region/--user-country/--timezone ..]` → `query, model, answer, citations[]` (title, url, snippet), `usage{tokens}`. **This is the "ask ChatGPT" surface** for the AEO test — the right tool because you're explicitly asking what AI assistants answer.
  * **Model:** use `gpt-5.4-mini` (the default). The other accepted values are `gpt-5.5` and `gpt-5.4-nano`. We are not using Gemini or Perplexity here.
  * Set `--user-country` (and `--city`/`--region`/`--timezone` if relevant) to match the brand's geo — see [Localization](#localization).
  * Check whether the prospect / competitor domains appear in `citations`, and tally cited domains across all queries.

***

## Localization

Derive geo from the brand brief (`brand-brief.md`). If the brand serves one dominant
geography, set it consistently on **every** geo-aware call: `seo` `--location-code` /
`--language-code`, `google` `--country` / `--language` / `--location`, `chatgpt`
`--user-country`. If the brand is global / multi-region, default to US/English
(`--location-code 2840` / `--language-code en`, `--country us`) and say so. Keep one geo
decision for the whole audit so numbers are comparable.

Common market codes (`--location-code` is DataForSEO numeric; `--language-code` is
two-letter): US `2840`, UK `2826`, Canada `2124`, Australia `2036`, Germany `2276`,
France `2250`, Poland `2616`, Netherlands `2528`, India `2356`, Singapore `2702`.
Languages: `en, de, fr, pl, nl, es`. `google` also accepts `--country` (ISO alpha-2 like
`us`, `de`) and `--location` (a canonical string like `"Austin,Texas,United States"`).

***

## What `dogfu` does NOT cover

The `dogfu` CLI exposes only the groups above for data (plus `crm`, `apollo`, `linkedin`,
`x` for the sales pipeline). For this audit that means:

* **No Bing / Trends / Reddit / Maps / backlink-marketplace commands.** If a report section genuinely needs one of these (e.g. a Reddit citation gap, local reviews), reach for your own **WebSearch / web-fetch tools** — don't invent a `dogfu` command for it. Keep it to what the report will actually use.
* **No brand-profile or report-publish under "data"** — brand context is a local file you build (`brand-brief.md`, see `SKILL.md`), and publishing is `dogfu report publish` (see `report.md`).
