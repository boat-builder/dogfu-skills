# Pipeline detail — interpretation manual

This is the *how to interpret* companion to SKILL.md. SKILL.md tells you which `dogfu`
command to run; this file tells you what the result means and how to judge fit. Read it
before running Stage B.

## 0. Inputs — assume almost nothing is given; derive the rest

The input is usually little more than a **target** — a company name, domain, person, or
LinkedIn/X URL, possibly buried in messy text. Do not expect any other field. Two things
*might* come from the prompt; everything else you derive:

- **Target** — extract it, then resolve to a single canonical company + root domain. If given a person, resolve to the company they currently work at. This is step one of Stage A.
- **ICP definition** — the qualification yardstick. The skill **ships its ICPs in `icps/`**; use the default unless the caller supplied their own or named a specific one (see "The ICP" in `SKILL.md`). Judge against it. Only if `icps/` is empty and none was supplied, ask for one before qualifying; never invent or assume one.

You must **derive** (do not wait to be told): business model / segment; geography /
language / audience; competitors; and the goal (honor a hinted goal like "just qualify"
or "draft DM angles"; otherwise run the full pipeline).

***

## Stage A — Discover

**Goal:** turn a sparse target into a structured map of the company and the people who
own the SEO/AEO decision. Gather facts; don't yet judge.

- **A1 — Resolve the company** to a canonical company + root domain. Every later phase keys off the domain.
- **A2 — Read the company's own pages** (homepage, About, product, pricing, blog) with your web-fetch tool. This is where you derive what the prompt didn't give you: what they do and their **problem statement** (reused to find competitors in B6); **business model / segment** (sets every SEO threshold downstream); **target audience & geography/language**; apparent stage/size and positioning. About + pricing are the richest. Record what you concluded and the evidence.
- **A3 — Map the social footprint:** company LinkedIn (description, headcount, employees) and company X/Twitter (does one exist, is it active — absence is a signal). If not found immediately, search before concluding it doesn't exist.
- **A4 — Identify the decision-maker:** who owns marketing / SEO / AEO — typically a founder, a marketing/growth lead, or both. Capture both when both plausibly own it.
- **A5 — Find the decision-maker's personal profiles:** both LinkedIn and X. Use LinkedIn detail + Google/AI search to find the X handle (LinkedIn won't give it directly). Record the URLs — they're the outreach handles you persist in Stage D, fit or not. Deep reading happens in Stage C.

**Stage A output:** resolved company + domain; what they do / who they serve; company
LinkedIn + X; relevant people; decision-maker(s) with LinkedIn + X URLs; any profile you
searched for but could not find (note the gap).

***

## Stage B — Qualify (ICP fit + SEO/AEO profile)

Run the phases in order; **stop early and mark "not in ICP"** the moment cheap signals
clearly disqualify (e.g. effectively zero organic footprint when the ICP requires "already
doing real SEO"). Calibrate every threshold to the **segment** and to the target's
**competitors**, never to absolute numbers. The runtime ICP is the yardstick — map each
phase's evidence back to its specific criteria.

### B0 — Competitive-conflict gate (run first; the cheapest disqualifier)
Before any paid SEO call, judge what Stage A (A2) told you about the business against one
question: **do they provide SEO/AEO capability to others, or merely use it themselves?**
- *Use it for their own growth* → ICP candidate; continue.
- *Build / sell / market it to others* → **competitor; exclude.**

This is deliberately broad — include adjacent cases: other AI SEO/AEO tools or agencies; AI
content / "marketing agent" tools positioned for ranking or answer-engine visibility; and
**general AI automation / agent-builder platforms that offer or market SEO/AEO agents or
content-automation use cases**, even when SEO/AEO isn't their headline. If the platform
produces SEO/AEO outcomes for its users, it overlaps with Berlin. Look for product / pricing
/ marketing language like "AI SEO", "AEO/GEO", "LLM visibility", "content automation for
ranking", "autonomous SEO", or SEO/AEO agents.

If it's a conflict: stop here (don't spend on B1+), verdict = **excluded (competitor)**, and
go straight to Stage D to record it. **Don't over-exclude:** a SaaS doing its own SEO is the
ICP, not a competitor — only exclude when they serve SEO/AEO capability to others; when truly
unsure, flag for a human rather than auto-excluding a strong fit.

### B1 — Footprint & scale
`site:<domain>` count; fetch `/sitemap.xml` and `/llms.txt`. Page volume is a proxy for
content investment, calibrated by segment (docs-heavy dev tools and ecommerce run large;
boutique B2B SaaS runs lean). A bare `site:rootdomain` count is often dominated by
subdomains and can understate the real footprint ~10× — corroborate with the keyword count
from B2 and a narrower `site:www.<domain>/blog` query; treat as a range. `llms.txt` present
= AEO-aware (a strong positive for an AI-using ICP).

### B2 — Organic outcomes
`domain-overview`: est. monthly organic traffic and traffic value (`etv`), total ranking
keywords (`count`), organic vs paid split. The single best "are they really doing SEO"
signal — outcomes, not vanity counts. Big footprint (B1) but low traffic/keywords = volume
without results (thin/programmatic content or a neglected site) — flag the divergence.

### B3 — Momentum / trend
`historical-rank-overview`: trajectory of traffic and keyword count. Rising = active
investment (strongest "live deal" signal); flat = maintenance; declining = recovery/
replatform need. Trend often matters more than absolute level.

### B4 — Ranking quality
`ranked-keywords` (top ~100 by volume): position distribution (top-3 / top-10 / 11+),
branded vs non-branded share, intent mix. High non-branded + many top-10 commercial terms
= mature, revenue-relevant SEO. Mostly branded = weak organic acquisition. Lots of
informational top-of-funnel content = content-led motion (common in an AI-using SaaS ICP).

### B5 — Technical health & stack
`lighthouse` (Core Web Vitals, performance/accessibility/SEO scores; scores are 0–1, ×100
to display) + `technologies` (CMS, analytics, SEO/AEO tooling). Strong CWV + a real SEO
stack (headless CMS, schema tooling, an SEO platform) = a literate, resourced operator.
Poor CWV = a concrete pain point to lead with. AEO-oriented tech (schema, structured data,
llms.txt tooling) = forward-looking. **AI-tooling signals** directly test an ICP's "AI
usage" criterion — note any. (`technologies` also returns `domain_rank`, a large integer
of ambiguous scale — use it only as a coarse *relative* hint, never as a grade.)

### B6 — Competitive gap (competitors you *discover*)
Start from the problem statement (A2). Find competitors two ways and merge: (1) search
buyer-intent keywords/prompts a buyer with that problem, in that geography, serving that
audience would type — see who consistently ranks/appears; (2) corroborate with an AI
answer engine (`google ai-mode` and `chatgpt search`) for "top alternatives and
competitors to <company> for <segment/audience>" and take named competitors from the
cited sources. Keep only same segment + audience + geography; drop keyword-only matches.
Do **not** use any SEO provider's "competitors" endpoint. Then benchmark: run B1/B2 on
each — `seo bulk-traffic-estimation` is efficient for many domains at once. Express the
target's traffic / keyword count as a **ratio** to the segment leader and median ("30% of
the leader's traffic"). Ratios beat absolute numbers and the gap is the size of the
opportunity you'd pitch.

### B7 — AEO / LLM visibility
Query AI answer engines (`google ai-mode`, `chatgpt search`) on 2–3 high-intent queries
the target should win in its category; check whether `<domain>` appears in the citations.
Cited + has `llms.txt` (B1) = AEO-aware, ahead of the curve. Absent despite decent classic
SEO = a clear, timely gap to lead with.

### B8 — Buyer literacy (light pass)
A quick scan of the decision-maker(s): do their public profiles/posts suggest they
personally understand SEO/AEO? (Full reading is Stage C.) A founder/marketing lead who
posts about SEO/AEO/content = a literate buyer, a strong fit signal. No evidence = neutral,
don't penalize.

### Synthesizing the SEO/AEO profile

Combine phases; never score on one metric. Look for agreement and divergence.

| Dimension | Evidence | Reading |
| :-- | :-- | :-- |
| Content footprint | B1 | scale of content investment |
| Organic outcomes | B2 | is the SEO working |
| Momentum | B3 | investing now vs coasting |
| Ranking quality | B4 | earned, defensible, revenue-relevant |
| Technical & stack | B5 | operator maturity |
| Competitive standing | B6 | gap vs leaders |
| AEO visibility | B7 | future-readiness |
| Buyer literacy | B8 / Stage A | can they operate a platform |

**Investment tier (synthesize, don't average):** *Heavy & effective* (strong B2 + rising
B3); *Heavy but inefficient* (large B1, weak B2 — a coaching/tooling opportunity); *Lean &
effective* (modest footprint, high traffic-per-page); *Light / nascent* (little footprint,
low traffic); *Dormant* (past traffic now declining, stale).

**Divergence checks — the most diagnostic part; call them out:** big footprint + low
traffic → thin/programmatic; high traffic + mostly branded → weak organic acquisition;
good classic SEO + absent from AI answers → AEO gap; strong historical peak + current
decline → recovery opportunity.

### ICP fit verdict
Map the profile back to the runtime ICP. State **strong / partial / weak (= not in ICP)**
with the specific signals that drove it. A **competitive conflict (B0) overrides everything**:
mark **excluded (competitor)** even when the behavioral fit is perfect — we don't reach out to
competitors. Strong/partial → Stage C. Weak → skip Stage C, go
to Stage D (still record the result and the reason).

**Revenue / size criteria, if the ICP has one.** `dogfu` exposes **no MRR/revenue
signal** — only headcount and funding (via `linkedin companies`) and whatever customer
counts the company publishes. So a revenue band is not directly measurable: infer size
loosely from those proxies. Unless the caller's ICP says size is a hard cutoff, treat it
as a **soft guide** — a behaviorally strong lead that looks larger (or smaller) than the
band is still a fit; **flag the size caveat prominently** rather than down-ranking it, and
let the human decide. Don't silently disqualify a behaviorally perfect lead for size.

***

## Stage C — Enrich (fit only)

Run only if strong/partial — this is the expensive deep read. Read each decision-maker on
**both LinkedIn and X**:
- **Background / history** — what they've built, prior companies/roles — reveals what they care about and how to speak to them.
- **Recent posts** — their latest posts reveal current interests and what's top of mind — the freshest, most personalizable signal.
- **Topics & voice** — what they post about and how they talk.

Then **pull the message hooks**: *personal hooks* (a recent post, a shared interest, a
milestone, a take they've voiced); *company hooks* (the specific SEO/AEO gaps from Stage B,
framed as the opening, not a critique); *relevance bridge* (why Berlin specifically maps to
their situation, in their language). **Note what's missing** — unfound profiles, thin
hooks, unknowns — so the DM step knows what's solid vs inferred.

**Stage C output:** per decision-maker — background summary, recent-interest summary, and a
short list of ready-to-use DM hooks (personal + company), each tied to its evidence.

***

## Stage D — CRM write (always — and save everything)

Persist every run — qualified or not — so work isn't lost, a "no" isn't re-prospected, and
**the outreach material you paid to gather is kept**. The research costs real money and
effort; none of it should be discarded just because a lead didn't fit. Upsert on the
resolved domain (search before create). Map this record onto Close:

Put each piece of data **where it belongs in the Close UI** — don't dump everything in one place:

- **Lead** (company): name, url (domain), a **brief** description (1–2 lines: segment + verdict + the single lead-with angle — keep the lead view scannable), and status (from verdict: Qualified / Bad Fit / Potential).
- **Curated lead fields** (set the ones you have, via the named flags — these are the *only* custom fields to fill): `--employees` (headcount), `--business-model` (e.g. SaaS), `--industry` (closest Close choice, e.g. Software), `--seo-pages` (B1 footprint), `--revenue` (USD, only if actually found). Skip any you don't have; don't pass blanks.
- **Contacts** (one per person you found — *always, fit or not*): name, title, role-in-decision, and **every profile URL (LinkedIn, X) in the native contact `urls` field** via repeated `-u` flags (plus `-e` email / `-p` phone if found). The native field renders on the contact card, so Sherin can message them straight from Close — don't leave these links only in prose.
- **Note** (the depth — "Notes & summaries"): segment / business model, market used, what they do / who they serve, the **company LinkedIn + X links** (these have no native lead field), **ICP-fit verdict**, **investment tier**, key metrics (page footprint, organic traffic est., traffic value est., # keywords, `llms.txt` y/n, AI-answer visibility y/n), competitive gap as ratios, reason for the verdict, date evaluated; and — for fits — background + recent-interest summaries, DM hooks (personal + company) with evidence, and known gaps.

**Custom fields:** Close supports lead/contact custom fields via its API, but `dogfu`
can't set them yet — so use the native fields above and the note. If dedicated custom
fields are wanted (e.g. a "Company LinkedIn" field on the lead), that needs a small `dogfu`
extension; flag it rather than calling the Close API directly.

**Disqualified leads:** still write the lead, verdict, and reason; set status **Bad Fit** so
they're excluded from outreach but retained and not re-researched next pass — **and still
save all contacts, their profile links (native field), and everything else you found.**

**Competitors (B0 conflict):** record them too — set status **Bad Fit** (there's no dedicated
"Competitor" status) and make the description and note say plainly **"COMPETITOR — do not
contact"**, with what they build/sell that conflicts. This keeps them out of outreach and off
the re-prospecting list. (Because B0 stops the run early, you'll have little SEO data to save —
that's fine; persist what you have.)

***

## Dynamic calibration (do this, don't skip it)

- **Segment sets the baseline.** Page-count, traffic, and keyword norms differ 10–100× across segments. Anchor to the derived business model, not a universal threshold.
- **Competitors set the yardstick.** Express the target's numbers as ratios to discovered competitors (B6). Relative standing is robust to estimation error; absolute numbers are not.
- **Geography sets the market.** Query SEO data for the prompt's market via `--location-code`/`--language-code`. A global SaaS may warrant more than one market.
- **Let outcomes overrule vanity.** When footprint (B1) and outcomes (B2) disagree, weight outcomes. Page count is a supporting proxy, not the verdict.
- **The ICP is supplied at runtime.** Re-read it each run and judge against it.
