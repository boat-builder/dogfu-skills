# Brand Category & Narrative Coherence — requirement spec

**Status:** draft · **Delivery:** parked (lead-research vs first-audit vs new skill — decided later)

## 1. What this is

AI assistants recommend a brand when every source they read tells the **same story about
what category the brand is in and why it belongs there**. This capability measures that:
we pull the brand's story from every place the story is told, compare the versions, and
report **where they agree, where they diverge, and why each divergence costs the brand
AI selection**.

It is not a technical-SEO check and not an "AI visibility" counter. It answers one
question: *is the brand's category story coherent across the surfaces an AI reads?*

## 2. The unit of comparison: the canonical claim

From every source we extract the **same structured claim**, so versions are directly
comparable:

| Field | Meaning |
| :-- | :-- |
| `category` | what the source files the brand under (primary + alternates) |
| `icp` | who it's for |
| `problem` | the problem it solves |
| `differentiation` | why it belongs / what sets it apart |
| `proof` | evidence the source cites (customers, numbers, integrations) |

## 3. The axes — where the story is told

| Axis | The story as told by… | Where we read it |
| :-- | :-- | :-- |
| **Owned** | the brand itself | website (home / about / pricing / product), llms.txt |
| **Founder / exec voice** *(optional)* | the founder(s) | LinkedIn / X profiles + posts of CEO (+ at most one co-founder). **Skip-if-silent:** no public narrative ≠ finding; extract only category/positioning claims, ignore the rest of the feed |
| **Third-party** | the market | **discovered per brand — see §4.** Review platforms, directories, press, analysts, communities |
| **Customer voice** | the customers | review bodies, testimonials, case studies on the discovered surfaces |
| **AI answers** | the model | ChatGPT + Google AI Mode answers about the brand and its category, plus which domains those answers cite |
| **Demand** *(light)* | the searching public | search volume on the claimed category terms — is the claimed category a real, searched category? |

Founder is deliberately its own axis, never merged into Owned: its diagnostic value is
precisely that it can *disagree* with the website (founder announces the new category on
LinkedIn while the site still sells the old one — a classic, high-value divergence).

## 4. Source discovery — the third-party panel is found, not hardcoded

A fixed list (G2, Capterra, …) is wrong per industry, vertical, and geo: dev-tools live
on GitHub/HN/Reddit, agencies on Clutch, DTC on Trustpilot, a German mid-market brand on
OMR Reviews and regional press. So the agent **discovers the panel per brand**, letting
Google and the AI answers do the ranking work — they are already vertical- and
geo-localized. Three discovery probes, then a union:

1. **Brand-anchored search** — `"<brand name>"` (quoted) and `"<brand name>" -site:<domain>`,
   in the brand's geo/language. The third-party sites ranking for the brand's own name
   are the surfaces where the market's version of the story already lives.
2. **Category-anchored search** — `best <category>`, `<category> reviews / tools`,
   `<top competitor> alternatives`. The sites dominating these SERPs are where **category
   verdicts are formed** — whether or not the brand appears on them.
3. **AI-citation harvest** — the domains the ChatGPT / AI Mode answers actually cite
   (already collected in first-audit Phase D). This is literally the list of sources the
   model trusts for this category; the strongest signal of the three.

**Panel selection:** rank the union; a site surfacing in **≥2 probes** is in the panel.
Cap at ~5–8 sources; drop thin stubs (an empty directory listing carries no story).
Localization follows the brand's geo on every probe (resolved once up front — see §7
step 0), so the panel self-localizes for free: the SERP and AI answers are already
vertical- and geo-ranked, so dev-tools surface GitHub/HN, agencies surface Clutch, a
German brand surfaces OMR — without us maintaining any per-vertical list.

**Absence is a finding, not a failure:** if fewer than ~2 substantive third-party sources
exist, that *is* the result — "the market barely writes about you" (an authority gap in
the claimed category), reported as such rather than padded with weak sources.

The same discovered panel serves the **Customer-voice** axis: review bodies and case
studies are read from the panel sources, not from a second discovery pass.

## 5. The comparison

1. **Extract** the canonical claim (§2) from each axis — one extraction pass per axis over
   the discovered/collected material.
2. **Normalize** categories to a shared taxonomy so "sales engagement" vs "revenue
   intelligence" compare as *different categories*, not merely different strings.
3. **Judge coherence** across the claims → outputs in §6.
4. **Substantiate** — when the story diverges or isn't landing, check *why*:
   - is the category **explicit** on the site, or only inferable?
   - is there **BOFU content in-category** (comparison / pricing / "vs" / use-case pages)?
   - is there an **authority gap in-category** — owned-vs-third-party citation share in AI
     answers, backlink authority vs competitors?
   - is the brand **present on the panel sources the AI actually cites** (probe 3)?

## 6. Output — every finding carries its reasoning

Findings ship as a **reasoning triple**; a bare score or verdict is never emitted:

```
claim    : the specific divergence or alignment
evidence : verbatim quote + URL from each side of the comparison
why      : one line on why this costs (or wins) AI selection
```

Deliverables:

- **Coherence score (0–100)** — with the triples that produced it, traceable end-to-end.
- **Category-placement matrix** — what each axis calls the brand, side by side.
- **Divergence map** — each mismatch as a triple, e.g. *site: "revenue-intelligence
  platform" (home hero) · G2: "Sales Engagement" (category page) · ChatGPT: "a CRM
  add-on" (cited from …) — an AI won't recommend you for a category its trusted sources
  don't place you in.*
- **Substantiation gaps** (§5.4) — prioritized story fixes, not tech fixes.
- **Panel report** — which sources were discovered, which were used, which were dropped
  and why (keeps the discovery honest and auditable).

## 7. Tooling & execution

Agent-neutral: this describes tool *classes*, not any one runtime. Three classes, chosen by
**what the source is**, never by preference.

- **Class A — structured pulls via the `dogfu` CLI.** Use when we need geo/language control,
  ranked & structured results, or an auth-walled source. Covers SERP + AI-answer discovery,
  socials, SEO/traffic/authority, and the CRM prior.
- **Class B — the agent's native web reading** (its built-in fetch/read). Use for open-web
  pages `dogfu` doesn't model: the discovered review/press/employer pages, case studies, and
  the brand's own site.
- **Class C — agent judgment (LLM).** Claim extraction, category normalization, the coherence
  judge. No API — this is the reasoning layer.

**Step 0 — resolve geo + vertical first (gates every probe).** Localization changes the whole
third-party panel, so fix it before searching: `country_iso_code` from `dogfu seo
technologies`, `industries`/location from `dogfu linkedin companies`, and the brand brief →
one `--country` / `--location-code` / `--language-code` decision applied to every probe (same
policy as first-audit). The same `technologies` call returns `social_graph_urls[]` — free
brand-handle discovery for the Owned/Founder axes.

| Step / axis | Class | Mechanism |
| :-- | :-- | :-- |
| Discovery probe 1 — brand-anchored SERP | A | `dogfu google search` on `"<brand>"` and `"<brand>" -site:<domain>` (+ `total_results` = the `site:` footprint) |
| Discovery probe 2 — category SERP dominance | A | `dogfu google search` on `best <category>`, `<category> reviews`, `<competitor> alternatives` |
| Discovery probe 3 — AI-citation harvest | A | citations from `dogfu google ai-mode` + `dogfu chatgpt search` (already run in first-audit Phase D) |
| Owned | B | native read of home / about / pricing / product (+ `llms.txt`) |
| Third-party + Customer | B | native read of the discovered panel pages (G2 / Capterra / Clutch / Trustpilot / press / Glassdoor + review bodies & case studies) |
| Founder / exec | A | `dogfu linkedin profiles` + `linkedin posts`, `dogfu x profiles` + `x posts` |
| AI answers | A | `dogfu google ai-mode` + `dogfu chatgpt search` (shared with probe 3) |
| Demand | A | `dogfu seo keyword-ideas` / `domain-overview` on the category terms |
| Extract · normalize · judge | C | agent reasoning over the gathered material |

**The three mechanism decisions:**

- **`site:` / "which sites rank" → `dogfu google search` (Class A), not native search.** We need
  a **geo-localized, ranked, structured** `results[]` plus the `total_results` footprint —
  native agent search can't reliably localize or return positions, so it can't *drive
  discovery*. Native reading (Class B) is for reading the pages discovery finds, not for
  ranking them.
- **Socials → `dogfu` (Class A), always.** LinkedIn/X block generic fetch, so the `linkedin` /
  `x` groups are the only reliable path. Find the founder from the lead-research
  decision-maker (Apollo `org enrich --with-people` → `linkedin_url`), or from `dogfu linkedin
  companies` / a name search, then pull profile + recent posts. Skip-if-silent still applies.
- **Random websites → the agent's native reading (Class B).** `dogfu` has no generic fetch;
  first-audit already routes every non-`dogfu` page to the agent's own web tools.

**Auth-wall fallback.** If a Class-B fetch is blocked (paywall / login / bot-wall — some
review sites, Glassdoor), fall back to the `title` + `snippet` the probe-1/2 SERP already
returned; if still thin, mark the source **unreadable** rather than guessing. Founder pulls
never hit this — they're Class A.

**Category normalization has no endpoint.** `dogfu linkedin companies.industries` and Close's
validated `--industry` give coarse *industry* buckets to seed it, but product-category
("revenue intelligence") normalization stays a Class-C judge task (see §9).

## 8. Cost discipline — every probe does double duty

The discovery searches are not overhead on top of data collection — they **are** data
collection: the SERP that reveals G2 is in the panel also hands you the G2 URL to read.
Clubbed with existing skills:

| Already paid for elsewhere | Reused here |
| :-- | :-- |
| lead-research discovery/qualification searches | brand-anchored probe (1) partially covered |
| first-audit brand brief | Owned-axis claim |
| first-audit Phase D AI answers + cited domains | AI-answers axis **and** discovery probe (3) |
| first-audit backlinks / crawl | authority + BOFU substantiation (§5.4) |
| first-audit keyword data | Demand axis |

Net-new cost is confined to: the SERP discovery probes (Class A, cheap), panel reading
(Class B, ~5–8 native fetches — near-free), the founder pull (Class A, skip-gated), and the
Class-C judge pass. The metered `seo` / AI-answer calls are the ones already run by the host
skill, so clubbing here is where the savings live.

## 9. Delivery (decided) & open questions

**Delivery — decided.** A dedicated **`narrative-audit`** skill owns the **canonical runtime
playbook** (`references/brand-narrative-coherence.md`). **`first-audit`** (mode `full` → a report
section) and **`lead-research`** (mode `light` → a CRM-note verdict) run the same check via a
**sub-agent** — but **only on an explicit user request**, with a token-minimal opt-in block in
each `SKILL.md`. The playbook is **duplicated** into each skill's `references/` (skills are
self-contained — no cross-skill include; `cp` from the canonical, documented in the
narrative-audit README). This design doc holds the rationale and stays out of the skills.

Open:

- **Taxonomy source** — curated category list vs letting the judge normalize ad hoc
  (consistency vs maintenance).
- **Monitoring mode** (later) — re-run on cadence, track **category drift** in the AI's
  placement of the brand as the leading indicator that the story is/isn't taking hold.
