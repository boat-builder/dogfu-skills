# Pipeline detail — interpretation manual

The *how to interpret* companion to SKILL.md. SKILL.md says which `dogfu` command
fulfils each step; this file says what the result means, how to compose the checkpoint
brief, and which datum fills which CRM field. Read it before running the Scout.

## Inputs — assume almost nothing is given; derive the rest

The input is usually just a **target** — a company name, domain, person, or LinkedIn/X
URL, possibly buried in messy text. Extract it, then resolve to a single canonical
company + root domain (a person resolves to the company they currently work at). You
must **derive** everything else: business model / segment; geography / language /
audience; competitors; and the goal (honor a hinted goal like "just scout it" or "draft
DM angles"; otherwise run the full flow).

***

## Scout

**Goal:** enough signal for the user to decide pursue or drop — at minimum spend.
Gather facts and compose the brief; the judging is the user's.

### S1 — Resolve the company
Canonical company + root domain. Every later call keys off the domain.

### S2 — Read the company's own pages
Homepage, About, product, pricing, blog — with your web-fetch tool. Derive: what they
do and their **problem statement** (reused for competitor discovery); **business model
/ segment** (sets every SEO threshold downstream); **target audience and
geography/language** (sets the SEO market codes); apparent stage and positioning;
**founder-led or professionally-managed** (who would sign). About + pricing are the
richest pages.

### S3 — Competitor check (the gate — run before any paid call)
From S2, one question: do they provide SEO/AEO capability **to others**, or merely use
it themselves? Conflict → stop the paid calls, brief the user with a drop
recommendation, record after confirmation. The full breadth of "competitor" (adjacent
AI-content/agent platforms included) is defined in SKILL.md. When genuinely unsure,
put the ambiguity in the brief instead of deciding.

### S4 — Map the social footprint
Company LinkedIn (`linkedin companies`: description, headcount, funding) and company
X. Search before concluding a profile doesn't exist; absence is itself a signal.
These URLs fill `--company-linkedin` / `--company-x`.

### S5 — Identify the decision-maker(s)
Who owns marketing / SEO / AEO — typically a founder, a marketing/growth lead, or
both; capture both when both plausibly own it. Find each person's LinkedIn **and** X
URLs (X handles come from Google/AI search, not LinkedIn). Deep reading waits for the
deep dive; a quick scan of their public posts for SEO/AEO literacy is a cheap, strong
signal worth one line in the brief.

### S6 — Footprint & scale
`site:<domain>` count; fetch `/sitemap.xml` and `/llms.txt`. Page volume proxies
content investment, calibrated by segment (docs-heavy dev tools and ecommerce run
large; boutique B2B SaaS runs lean). A bare `site:` count is a rough estimate often
dominated by subdomains and can understate ~10× — corroborate with S7's keyword count
and a narrower `site:www.<domain>/blog`, and treat it as a range. `llms.txt` present =
AEO-aware (worth a flag in the brief).

### S7 — Organic outcomes
`domain-overview`: ranking-keyword count (`organic.count`), estimated traffic value
(`etv`), organic vs paid split. The single best "are they really doing SEO" signal —
outcomes, not vanity counts. Big footprint (S6) + low outcomes = volume without
results (thin/programmatic content); flag the divergence.

### S8 — Momentum
`historical-rank-overview`: trajectory of keywords and traffic. **Rising** = active
investment (the strongest "live deal" signal); **flat** = maintenance; **declining** =
recovery/replatform need. Trend often matters more than absolute level.

### S9 — Firmographics
`apollo org enrich --domain <d> --with-people` (~1 credit): estimated
`annual_revenue`, `employee_count`, `marketing_headcount`, `total_funding`,
`latest_funding_round`, `founded_year`, plus a free `people[]` list to corroborate S5.
Revenue is a modeled estimate — coarse for small private SaaS — so the brief must say
the source and confidence, and cross-check against headcount and funding (a "$40M ARR"
read on a 6-person seed company is the model hallucinating). If Apollo isn't connected
(`412`), fall back to proxies: LinkedIn headcount + funding, pricing page, published
customer counts — and mark ARR "unknown (proxies only)".

### Synthesis — the two derived reads

**SEO investment tier** (fills `--seo-investment-tier`; synthesize, don't average):
*Heavy & effective* (strong S7 + rising S8) · *Heavy but inefficient* (large S6, weak
S7) · *Lean & effective* (modest footprint, high traffic-per-page) · *Light / nascent*
(little footprint, low outcomes) · *Dormant* (past peak, now declining/stale).

**Divergences** — the most diagnostic part; they are the "read" line of the brief: big
footprint + low traffic → thin/programmatic; strong history + current decline →
recovery opportunity; good outcomes + tiny team → efficient operator (great fit
signal); large team + weak outcomes → coaching/tooling opportunity.

### Composing the brief

The format is fixed in SKILL.md. Filling it well:

- **ARR line:** value + source + confidence + band position ("~$4M (Apollo, coarse) —
  in band" / "unknown — proxies: 12 heads, seed-funded, likely <$5M"). Near the band
  edge, say so neutrally; the soft-edge policy is the user's to apply.
- **SEO motion line:** pages · keywords · traffic value · momentum arrow. If motion is
  effectively zero, that's a flag, not a silent drop.
- **Flags:** competitor conflict (or ambiguity), far-out-of-band size, no SEO motion,
  no in-house team, unknown ARR, anything implausible in the data.
- **Read (≤3 sentences):** strongest signal, weakest signal, what the deep dive would
  settle. A recommendation is welcome ("worth the deep dive" / "recommend drop:
  agency with no in-house motion") — as a recommendation.
- **Batch:** one row per lead, same columns, one consolidated ask. Sort so the
  clearest pursues are on top.

***

## Deep dive (only after the user says pursue)

**Goal:** the outreach-grade profile — what to say, to whom, with what evidence.

### D1 — Ranking quality & top pages
`ranked-keywords` (top ~100 by volume): position distribution (top-3 / top-10 / 11+),
branded vs non-branded share, intent mix. High non-branded + top-10 commercial terms =
mature, revenue-relevant SEO. Mostly branded = weak organic acquisition. Heavy
informational top-of-funnel = content-led motion. Pair with `relevant-pages` (top pages
by organic traffic — `page_address` + `etv`) to see *which* URLs actually pull the
traffic: a specific high-value page ("your `/guide/x` ranks #3 and drives ~N visits") is
concrete DM-hook color and shows where their SEO is really working.

### D2 — Technical health & stack
`lighthouse` (scores are 0–1; ×100 to display) + `technologies` (CMS, analytics, SEO
tooling). Strong CWV + a real SEO stack = a literate, resourced operator. Poor CWV = a
concrete pain point to lead with. AI/AEO-oriented tooling (schema, llms.txt tooling) =
forward-looking. `technologies` also returns `domain_rank` — a large integer of
ambiguous scale; use only as a coarse *relative* hint, never a grade.

### D3 — Competitive gap
From the S2 problem statement, find competitors two ways and merge: (1) search the
buyer-intent keywords a buyer with that problem, in that geography, would type — who
consistently ranks; (2) corroborate with AI engines (`google ai-mode`, `chatgpt
search`) asking for top alternatives to <company> for <segment/audience>. Keep only
same segment + audience + geography. Do **not** use an SEO provider's "competitors"
endpoint. Benchmark with `bulk-traffic-estimation` and express the target as a
**ratio** to the segment leader and median ("30% of the leader's traffic") — ratios
are robust to estimation error, and the gap is the size of the pitch.

### D4 — AEO / LLM visibility
2–3 high-intent queries the target should win, on `google ai-mode` + `chatgpt search`;
is `<domain>` in the citations? Cited + `llms.txt` (S6) = ahead of the curve. Absent
despite decent classic SEO = a clear, timely gap to lead with. Fills
`--aeo-visibility` (Cited / Partial / Absent).

### D5 — Decision-maker deep-read → DM hooks
Both LinkedIn and X, profile + recent posts (async scrapes — detached with `-o FILE`).
Pull: background (what they've built, prior roles), current interests (recent posts —
the freshest personalizable signal), topics & voice. Then compose the hooks:
*personal* (a recent post, a milestone, a voiced take), *company* (the D1–D4 gaps
framed as openings, not critiques), and the *relevance bridge* (why Berlin maps to
their situation, in their language). Note what's missing — unfound profiles, thin
hooks — so the DM step knows solid from inferred.

### D6 — Verified emails
`apollo people email` for the **1–2 people who will actually be contacted** (credits
per person). `--linkedin-url` matches best. The email goes on the contact (`-e`).

***

## CRM write — the attribute → source map

Statuses and placement rules are in SKILL.md. Fill **every flag you have a value
for; never pass a blank**. Values for choices fields must match Close's options (the
error echoes the allowed list; pick the closest or omit and note it).

| Flag | Source |
| :-- | :-- |
| `--industry` | S2 vertical, mapped to Close's choices |
| `--business-model` | S2 (SaaS / Marketplace / …) |
| `--primary-market` | S2 geography |
| `--year-founded` | S9 `founded_year` (or LinkedIn / site) |
| `--employees` | S4 LinkedIn `employee_count` or S9 `employee_count` |
| `--marketing-team-size` | S9 `marketing_headcount` |
| `--revenue` | S9 `annual_revenue` (estimate — the note says so) |
| `--funding-stage` | S9 `latest_funding_round` / S4 LinkedIn `funding.last_round_type`, mapped to Close's choices |
| `--total-funding` | S9 `total_funding` / S4 LinkedIn funding raised |
| `--seo-pages` | S6 `site:` estimate |
| `--organic-keywords` | S7 `organic.count` |
| `--seo-investment-tier` | Scout synthesis |
| `--seo-momentum` | S8 (Rising / Flat / Declining) |
| `--aeo-visibility` | D4 (leave unset if the deep dive didn't run) |
| `--company-linkedin` / `--company-x` | S4 |

The **note** carries the depth: what they do / who they serve, market used, metrics
with sources (labeled as estimates), the brief's read, the **user's decision and
reason**, date; and — pursued leads — the competitive ratios, DM hooks with evidence,
and known gaps. Contacts (every person found, any decision) carry name, title, LinkedIn
+ X in the native `urls` field, verified email when resolved.

***

## Calibration (do this, don't skip it)

- **Segment sets the baseline.** Page/traffic/keyword norms differ 10–100× across
  segments; anchor to the derived business model, never a universal threshold.
- **Competitors set the yardstick.** Ratios to discovered competitors beat absolute
  numbers everywhere they're available.
- **Geography sets the market.** Query SEO data with the derived market's
  `--location-code`/`--language-code`; a global SaaS may warrant two markets.
- **Outcomes overrule vanity.** When footprint (S6) and outcomes (S7) disagree, weight
  outcomes.
