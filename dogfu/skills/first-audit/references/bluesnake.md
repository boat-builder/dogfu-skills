# Bluesnake crawl (technical + on-page layer)

Bluesnake crawls the prospect's **public** site and exposes every crawled page in a
read-only **SQLite** database you query with SQL. This is the "pages"/technical layer of
the audit. All tools are MCP tools on the Bluesnake server.

## Lifecycle

1. **`create_project(main_domain="<domain>")`** → returns a `project_id`. Do this in Phase A3 so competitors can be attached. (A crawl can also run without a project; the project only exists so `project_comparison` can line the sites up.)
2. **`start_crawl(url="https://<domain>", profile="Default audit")`** → returns a `crawl_id` immediately and runs in the background. Spider mode (default) follows links from `url`. **Only one crawl runs at a time.** Profiles available: `Default audit`, `Parity audit (rendered)`, `Parity audit (text)` — the parity profiles compare raw vs JS-rendered HTML (extra AEO signal); `Default audit` is fine for a first audit.
   * You usually don't need a scope cap for a normal site. If a site turns out to be huge and the crawl is impractical, you can pass `config={"limits.max_urls": N}` (discover keys with `list_config_options`).
3. **`add_competitor(project_id, domain)`** then `start_crawl(url=competitor_root)` for each competitor — sequential, after the prospect's crawl.
4. **`crawl_status(crawl_id)`** → poll until `status == "completed"`. `list_crawls` shows all crawls (id, seed, status, URL count). Wait for completion unless the user says otherwise.
5. **`issue_summary(crawl_id)`** → the audit verdict. **`query(sql, crawl_id, max_rows)`** → arbitrary read-only SELECT/CTE. **`project_comparison(project_id, include_optional=true)`** → prospect-vs-competitor scorecard.

## Database schema (same shape for every crawl)

Run `get_database_schema()` once if you need the authoritative DDL. Key tables:

* **`pages`** — one row per URL. Columns: `url, scope` (internal|external), `state` (crawled|blocked\_robots|error|…), `status_code, indexable` (0/1), `indexability_status, response_time_ms, size, redirect_url, redirect_type, inlinks, link_score, unique_inlinks, unique_outlinks, closest_similarity, near_dup_count, duplicate_of`, and JSON columns `headers`, `structured`, `jsdiff`, `facts`.
  * `facts` (JSON, PascalCase keys): `Titles, Descriptions, H1s, H2s, MetaRobots, CanonicalHTML, HreflangHTML, WordCount, TextRatio, Flesch, Lang, Links`. Access with `json_extract(facts,'$.WordCount')`, `json_extract(facts,'$.Titles[0]')`, etc.
  * `structured` (JSON): schema markup, shape `{"formats":["jsonld"],"types":["Organization","FAQPage",...],"errors":["..."]}`.
  * `jsdiff` (JSON): static-vs-rendered differences (JS parity).
* **`issues`** — `(url, issue, detail)`; one row per occurrence. `issue_summary` maps `issue` ids → names/severities and counts distinct URLs.
* **`links`** — edge list (`src, dst, type, anchor, nofollow, rel, ...`).
* **`llmstxt`** **/** **`llmstxt_links`** — one row per fetched `/llms.txt`; `found` (0/1), `title, summary, malformed, content`. Empty table = the site has no llms.txt.
* **`sitemap_entries`**, `frontier`, `meta`, `analysis`.

## Query cookbook

Technical:

```sql
-- status-code mix (for a donut)
SELECT status_code, COUNT(*) n FROM pages WHERE scope='internal' GROUP BY status_code ORDER BY n DESC;
-- health snapshot
SELECT COUNT(*) internal_pages, SUM(indexable) indexable,
       SUM(CASE WHEN CAST(json_extract(facts,'$.WordCount') AS INT)<200 THEN 1 ELSE 0 END) thin_under_200w
FROM pages WHERE scope='internal' AND state='crawled';
-- weakest internal linking
SELECT url, inlinks, link_score FROM pages WHERE scope='internal' ORDER BY link_score ASC LIMIT 20;
-- affected URLs for any issue id from issue_summary
SELECT url, detail FROM issues WHERE issue='<id>';
```

AEO on-page (all also SEO-relevant — feature these prominently):

```sql
-- schema-type coverage (high-value AEO types: FAQPage, Article, HowTo, Product, Organization, BreadcrumbList)
SELECT je.value AS schema_type, COUNT(DISTINCT p.url) AS pages
FROM pages p, json_each(p.structured,'$.types') je
WHERE p.scope='internal' GROUP BY je.value ORDER BY pages DESC;
-- pages with NO schema, and pages with invalid rich-result markup
SELECT COUNT(*) FROM pages WHERE scope='internal' AND structured IN ('[]','null','');
SELECT url, json_extract(structured,'$.errors') FROM pages WHERE json_extract(structured,'$.errors') IS NOT NULL;
-- llms.txt presence
SELECT url, found, malformed, title FROM llmstxt;
```

Relevant `issue_summary` ids to surface for AEO + SEO: `structured_missing`,
`structured_validation_error`/`_warning`, `h1_missing`/`h1_duplicate`,
`description_missing`, `title_missing`/`title_duplicate`, the `js_*_updated` /
`js_canonical_mismatch` parity issues, and `content_low_word_count`.

## Why these matter for AEO

AI answer engines extract a page's topic and quotable facts from its title, meta, headings
and structured data, and many crawl without executing JavaScript. So missing/duplicate
H1s and metadata, absent or invalid schema/JSON-LD, content that only appears after JS
(`jsdiff` / `js_*` issues), thin pages, and a missing `llms.txt` all reduce how often and
how accurately the brand gets surfaced and cited — which is exactly what this audit measures.
