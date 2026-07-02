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
  → query, answer (markdown), citations[] (title, url, source, snippet). Use for AEO visibility + competitor discovery.

## `chatgpt`
- **`search --query "<q>" [--model gpt-5.4-mini] [--search-context-size low|medium|high] [--domain-filter <d>...] [--reasoning-effort ..] [--city/--region/--user-country/--timezone ..]`**
  → query, model, answer, citations[] (title, url, snippet), usage{tokens}. Second AI engine for B6/B7 corroboration.

## `seo`
- **`domain-overview --target <domain> [--location-code N] [--language-code xx]`** ← **B2.**
  → target, organic{count (≈ ranking-keyword count), etv (traffic value), paid_traffic_cost, is_new/up/down/lost}, paid{}. **No domain-rank field here** — for a coarse domain-strength hint use `domain_rank` from `technologies`.
- **`ranked-keywords --target <domain> [--location-code] [--language-code] [--limit 100] [--offset] [--order-by "keyword_data.keyword_info.search_volume,desc"] [--filters JSON]`** ← **B4.**
  → keyword, search_volume, cpc, competition, competition_level, rank, rank_group, url, title, intent, etv.
- **`keyword-ideas --keyword "<seed>"... [--location-code] [--language-code] [--limit] [--order-by]`**
  → keyword, search_volume, cpc, competition, competition_level, difficulty.
- **`historical-rank-overview --target <domain> [--location-code] [--language-code] [--date-from YYYY-MM-DD] [--date-to ..]`** ← **B3.**
  → RankHistoryPoint[]: year, month, organic{count, etv, pos_1, pos_2_3, pos_4_10, pos_11_plus, is_*}, paid.
- **`relevant-pages --target <domain> [--location-code] [--language-code] [--limit] [--order-by "metrics.organic.etv,desc"]`**
  → page_address, organic{count, etv, pos_*}, paid.
- **`technologies --target <domain>`** ← **B5 stack + domain-strength hint (`domain_rank`).**
  → target, domain_rank (**large integer, NOT 0–100; ambiguous scale — use only relatively**), last_visited, country_iso_code, emails[], phone_numbers[], social_graph_urls[], technologies{category:{subcategory:[tech,...]}}.
- **`lighthouse --url <url> [--strategy mobile|desktop] [--locale en-US]`** ← **B5 (~10–90s).**
  → performance_score, accessibility_score, seo_score, best_practices_score (0–1, ×100 to show), core_web_vitals{lcp_ms, fcp_ms, cls, tbt_ms, ...}, field_data (CrUX; absent for low-traffic sites), opportunities[], diagnostics[].
- **`bulk-traffic-estimation --target <d>... [--location-code] [--language-code] [--item-type organic ..]`** ← efficient **B6** benchmarking (up to 1,000 domains).
  → target, organic{count, etv}, paid.

## `crm` (Close)
- **`whoami`** → current user (id, name, email).
- **`status list`** → LeadStatus[] (id, label). Resolve IDs here; never hardcode.
- **Leads** (`crm lead ...`): `create -n <name> [-u url] [-d desc] [-s status_id] [curated flags]` · `list [-n name] [-q query] [-s status_id] [-l N] [--sort ..]` (no filter = newest-first; --name/--query/--status-id narrow it — this is also how you search) · `get <lead_id>` · `update <lead_id> [-n] [-u] [-d] [-s] [curated flags]` · `delete <lead_id> [-y]`. → Lead: id, name, url, description, status_id, status_label, **industry, employees, revenue, business_model, seo_pages**, contacts[].
  - **Curated custom-field flags** (on `create`/`update`): `--employees <n>`, `--revenue <usd>`, `--business-model <text>`, `--industry <choice>` (validated against Close's allowed list — closest match, e.g. SaaS→Software), `--seo-pages <n>`. Set only the ones you have; everything else → `-d`/note. These five are the *only* custom fields dogfu sets.
- **Contacts** (`crm contact ...`): `create <lead_id> -n <name> [-t title] [-e email]... [-p phone]... [-u url]...` · `list <lead_id>` · `get <contact_id>` · `update <contact_id> [-n] [-t]` · `delete <contact_id> [-y]`. **`-u`/`-e`/`-p` are native Close fields** (urls/emails/phones) — a person's LinkedIn/X go in `-u` and render on the contact card; repeatable.
- **Tasks** (`crm task ...`): `create -l <lead_id> -t <text> [-d due] [-a user]` · `list [-l lead_id] [-p]` · `update <task_id> ...` · `complete <task_id>` · `delete <task_id> [-y]`.
- **Notes** (`crm note ...`): `create <lead_id> -t "<text>"` · `list <lead_id> [-l N]`. ← put the rich qualification write-up + DM hooks here. **Note bodies are HTML-escaped** on write (`&`→`&amp;`, `<`/`>`/`'`→entities); use plain text, not literal `<`/`>`/`&` for structure.

**Fields & their proper home.** Native: lead `-u` = website; contact `-u` = a person's
LinkedIn/X (repeatable, renders on the card). Curated lead custom fields = the five flags
above (`--employees/--revenue/--business-model/--industry/--seo-pages`) — set what you have.
Everything else (company socials, verdict, metrics, hooks) → the lead description (brief) or
the note. dogfu exposes **no other** custom-field setters by design — don't try to set
fields outside the curated five.

### Upsert pattern
1. `crm status list` → pick the status_id for the verdict (Qualified / Bad Fit / Potential).
2. `crm lead list -n "<company>"` (or `-q` with the domain) → if a match exists, `lead update`; else `lead create` with a **brief** `-d` description **plus the curated flags you have** (`--employees`, `--business-model`, `--industry`, `--seo-pages`, `--revenue`).
3. `crm contact create <lead_id> -n "<name>" -t "<title>" -u <linkedin> -u <x> [-e <email>]` for each person (links in the native `urls` field; `-e` = the Apollo verified email for fits).
4. `crm note create <lead_id> -t "<structured profile + metrics + verdict + company links + hooks>"`.

## `apollo` (enrichment — Stage C, fits only)
Per-user Apollo key, connected in the Console → **Apollo Integration** (like the Close key).
A `412` "no Apollo API key configured" = not connected; surface it, don't retry.
**Credit-bearing — run only for ICP fits.** Identify a company by `--domain` (preferred) or `--name`.
- **`org enrich --domain <d> | --name <n> [--with-people/--no-people] [--title <t>...] [--seniority <s>...] [--limit N]`**
  → ApolloOrganization: id, name, domain, website_url, industry, employee_count, **annual_revenue** (modeled estimate — coarse for small private SaaS), **marketing_headcount**, total_funding, latest_funding_round, latest_funding_date, founded_year, linkedin_url, phone, technologies[], people[] (ApolloPerson: name, title, seniority, linkedin_url — **no email**). `--with-people` (default on) runs a *free* people search by title/seniority at the domain (`--limit` caps it, default 10); it needs a master key (else the org returns with `people[]` empty). ~1 Apollo credit.
- **`people email --linkedin-url <url> | --name "<full name>" [--first-name <f>] [--last-name <l>] [--domain <d>] [--organization-name <n>]`**
  → ApolloPerson[] (0 or 1): name, title, seniority, **email**, email_status (verified | likely to engage | unavailable), linkedin_url, organization_name, organization_domain. Resolves ONE person → verified work email; **consumes credits** (resolve only people you'll contact). Take the person from the `org enrich --with-people` list; `--linkedin-url` matches best, else `--name` + `--domain`. Best-match, not exact.
