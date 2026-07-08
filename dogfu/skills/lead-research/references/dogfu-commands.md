# `dogfu` command catalog

Read this once so you don't spend calls on `--help`. Once the dogfu MCP's
`get_setup_instructions` flow has installed and configured the CLI, run any command from
anywhere on your PATH:

```bash
dogfu <group> <cmd> [flags]
```

JSON to stdout by default. Use `-o FILE` to dump large payloads to a file and read back
only the fields you need. Output is always the canonical, normalized model.

Calls are independent HTTP requests — there's no single-process lock, so you may run them
concurrently. Cost discipline still applies: pick the cheapest informative call next.

**Discovery scrapes are slow + metered — never block a foreground shell on them.**
`linkedin posts --profile-url`, `linkedin profiles --name`, and `x posts --profile` trigger
an async scrape (~1–10 min) that bills on trigger even if you time out and get nothing. Run
detached with `-o FILE` and poll the file; prefer the fast `--url` path when you have the URL.

## Common output flags (all leaf commands)
- `-f, --format [json|table]` — default `json`. Use `json`.
- `-o, --output FILE` — write JSON to a file instead of stdout.

## Market codes (for `seo` and `google`)
`--location-code` is numeric (DataForSEO-style); `--language-code` is two-letter.
Common: US `2840`, UK `2826`, Canada `2124`, Australia `2036`, Germany `2276`,
France `2250`, Poland `2616`, Netherlands `2528`, India `2356`, Singapore `2702`.
Languages: `en`, `de`, `fr`, `pl`, `nl`, `es`. If you truly can't tell the market,
default to `2840` / `en` and say so. `google` also accepts `--country` (ISO alpha-2 like
`us`, `de`) and `--location` (canonical string like `"Austin,Texas,United States"`).

---

## `linkedin`
- **`profiles --url <u>... | --name "<Full Name>"...`** `[--limit-per-input N] [--wait-seconds 0-600]`
  Fetch by URL or discover by name. **`--name`: async scrape ~1–10 min; `--url` fast.** → id, name, headline, about, current_company, current_title, location, followers, connections, linkedin_url, experience[] (company, title, location, start_date, end_date, url), education[]. **Often sparse:** private profiles can return `experience: []` and a missing `current_title` — corroborate background from `linkedin posts` and a web search.
- **`companies --url <u>...`** `[--wait-seconds N]`
  → id, name, linkedin_url, website, domain, description, about, industries, company_size, employee_count, organization_type, followers, logo_url, funding (last_round_type/date/raised, rounds), investors[]. *(No full employee list.)*
- **`jobs --url <u>... | --keyword <k> [--location ..] [--company ..] ...`** → title, company, location, employment_type, seniority, posted_date, applicants, base_salary, summary, url.
- **`posts --url <u>... | --profile-url <p>...`** `[--limit-per-input N]`
  Posts by URL or recent posts by author. **`--profile-url`: async scrape ~1–10 min; `--url` fast.** → id, url, post_type, title, text, date_posted, likes, comments, author_name, author_url, hashtags[], embedded_links[].

## `x` (Twitter)
- **`posts --url <u>... | --profile <p> [--start-date YYYY-MM-DD] [--end-date ..]`** `[--limit-per-input N]`
  **`--profile`: async scrape ~1–10 min; `--url` fast.**
  → id, url, text, author, date_posted, likes, replies, reposts, quotes, views, is_repost, is_verified, hashtags[], photos[], external_url.
- **`profiles --url <u>...`** → id, name, url, biography, location, followers, following, posts_count, is_verified, is_business_account, date_joined, external_link, category.

## `google`
- **`search --query "<q>" [--country us] [--language en] [--location "..."] [--max-results 1-100]`**
  → query, total_results (Google's estimate — use for `site:` footprint), features{answer_box, related_searches, ...}, results[] (title, url, snippet, displayed_link, position).
- **`ai-mode --query "<q>" [--country] [--language] [--location]`**
  → query, answer (markdown), citations[] (title, url, source, snippet). Use for AEO visibility + competitor discovery (deep dive).

## `chatgpt`
- **`search --query "<q>" [--model gpt-5.4-mini] [--search-context-size low|medium|high] [--domain-filter <d>...] [--reasoning-effort ..] [--city/--region/--user-country/--timezone ..]`**
  → query, model, answer, citations[] (title, url, snippet), usage{tokens}. Second AI engine to corroborate competitor discovery + AEO visibility.

## `seo`
- **`domain-overview --target <domain> [--location-code N] [--language-code xx]`** ← **Scout: organic outcomes.**
  → target, organic{count (≈ ranking-keyword count), etv (traffic value), paid_traffic_cost, is_new/up/down/lost}, paid{}. **No domain-rank field here** — for a coarse domain-strength hint use `domain_rank` from `technologies`.
- **`ranked-keywords --target <domain> [--location-code] [--language-code] [--limit 100] [--offset] [--order-by "keyword_data.keyword_info.search_volume,desc"] [--filters JSON]`** ← **deep dive: ranking quality.**
  → keyword, search_volume, cpc, competition, competition_level, rank, rank_group, url, title, intent, etv.
- **`keyword-ideas --keyword "<seed>"... [--location-code] [--language-code] [--limit] [--order-by]`**
  → keyword, search_volume, cpc, competition, competition_level, difficulty.
- **`historical-rank-overview --target <domain> [--location-code] [--language-code] [--date-from YYYY-MM-DD] [--date-to ..]`** ← **Scout: momentum.**
  → RankHistoryPoint[]: year, month, organic{count, etv, pos_1, pos_2_3, pos_4_10, pos_11_plus, is_*}, paid.
- **`relevant-pages --target <domain> [--location-code] [--language-code] [--limit] [--order-by "metrics.organic.etv,desc"]`** ← **deep dive: top pages by traffic (DM-hook color).**
  → page_address, organic{count, etv, pos_*}, paid.
- **`technologies --target <domain>`** ← **deep dive: stack + domain-strength hint (`domain_rank`).**
  → target, domain_rank (**large integer, NOT 0–100; ambiguous scale — use only relatively**), last_visited, country_iso_code, emails[], phone_numbers[], social_graph_urls[], technologies{category:{subcategory:[tech,...]}}.
- **`lighthouse --url <url> [--strategy mobile|desktop] [--locale en-US]`** ← **deep dive: technical health (~10–90s).**
  → performance_score, accessibility_score, seo_score, best_practices_score (0–1, ×100 to show), core_web_vitals{lcp_ms, fcp_ms, cls, tbt_ms, ...}, field_data (CrUX; absent for low-traffic sites), opportunities[], diagnostics[].
- **`bulk-traffic-estimation --target <d>... [--location-code] [--language-code] [--item-type organic ..]`** ← efficient **deep-dive competitor benchmarking** (up to 1,000 domains).
  → target, organic{count, etv}, paid.

## `crm` (Close)
The CRM command surface is owned by the **crm** skill — read
**`../../crm/references/records.md`** (reads, record editing, the curated attribute
flags, where data belongs) and, for the post-checkpoint research write,
**`../../crm/references/intake.md`** (the decision → status contract and the one-pass
upsert). Quick anchors: resolve status ids with `dogfu crm status list`;
leads / contacts / notes / tasks live under `dogfu crm lead|contact|note|task …`.

## `apollo` (firmographics + verified emails)
Per-user Apollo key, connected in the Console → **Apollo Integration** (like the Close key).
A `412` "no Apollo API key configured" = not connected; surface it, don't retry.
Identify a company by `--domain` (preferred) or `--name`.
- **`org enrich --domain <d> | --name <n> [--with-people/--no-people] [--title <t>...] [--seniority <s>...] [--limit N]`** ← **Scout: firmographics (~1 credit).**
  → ApolloOrganization: id, name, domain, website_url, industry, employee_count, **annual_revenue** (modeled estimate — coarse for small private SaaS), **marketing_headcount**, total_funding, latest_funding_round, latest_funding_date, founded_year, linkedin_url, phone, technologies[], people[] (ApolloPerson: name, title, seniority, linkedin_url — **no email**). `--with-people` (default on) runs a *free* people search by title/seniority at the domain (`--limit` caps it, default 10); it needs a master key (else the org returns with `people[]` empty).
- **`people email --linkedin-url <url> | --name "<full name>" [--first-name <f>] [--last-name <l>] [--domain <d>] [--organization-name <n>]`** ← **deep dive only — consumes credits per person.**
  → ApolloPerson[] (0 or 1): name, title, seniority, **email**, email_status (verified | likely to engage | unavailable), linkedin_url, organization_name, organization_domain. Resolves ONE person → verified work email; resolve only the **1–2 people who will actually be contacted**, after the user's pursue. Take the person from the `org enrich --with-people` list; `--linkedin-url` matches best, else `--name` + `--domain`. Best-match, not exact.
