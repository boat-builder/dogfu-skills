---
name: lead-research
description: >-
  Run Agent Berlin's staged lead-research pipeline — Scout → checkpoint → Deep dive —
  using the `dogfu` CLI. Use this whenever the user hands you a sales target (a
  company name, a domain, a person, or a LinkedIn/X URL) and wants it researched, sized
  up, qualified, or enriched for outreach. Triggers include "research this lead",
  "qualify this prospect", "is this company a fit", "find the decision-maker and DM
  hooks", "prospect this domain", or just a bare LinkedIn/X/company link dropped in
  with intent to evaluate it. Also use it for batch prospecting. The skill gathers
  cheap signals first, presents a succinct decision brief, and the USER decides —
  pursue or drop — whether to spend on the deep dive; there is no auto-qualify, even
  when the data is clear-cut. This is the canonical way to do sales lead research at
  Berlin — prefer it over ad-hoc web searches.
---

# Lead Research (powered by `dogfu`)

Turn a sparse sales **target** into a researched company profile and a **human
decision**. You gather the cheap, high-signal facts first, compress them into a short
decision brief, and the user decides whether the lead is worth the expensive deep dive.
You research and recommend; **you never decide**.

This skill is the **orchestration layer**. The data comes from the `dogfu` CLI. Your
job is to run the right `dogfu` commands in the right order, read the JSON they return,
and reason over it. The stage-by-stage interpretation logic lives in
**`references/pipeline.md`** — read it before running the Scout.

## The flow

1. **Scout** *(always)* — resolve the target to a company + root domain, read its own
   pages and socials, find the decision-maker(s), then pull the **cheap** SEO and
   firmographic signals. Capture every company attribute you find.
2. **Checkpoint** *(always)* — present the decision brief (format below) and ask the
   user: **pursue or drop**. This is the only place spend escalates — nothing
   expensive runs until the user answers. Never skip it, never answer it yourself.
3. **Deep dive** *(pursue only)* — ranking quality, top pages, technical health,
   competitive gap, AEO visibility, decision-maker deep-reads, verified emails, DM hooks.

Cost logic: Scout calls are cheap (one `site:` search, one domain overview, one history
pull, one LinkedIn company read, ~1 Apollo credit). Deep-dive calls are the expensive
ones (keyword pulls, Lighthouse, competitor benchmarking, AI-answer queries, async
social scrapes, per-person email credits) — they run only after the user opts in.

## Who we are

Berlin — an AI SEO/AEO platform that replaces hiring an SEO agency or building a large
in-house SEO team. We redirect SEO / content / agency budget a company **already
spends**. Companies that *build or sell* SEO/AEO capability to others are competitors,
not customers.

## Competitive-conflict gate — check first, spend nothing on conflicts

As soon as the site read tells you what the company does, apply one test:

- **Do they USE SEO/AEO for their own growth?** → a prospect. Keep going.
- **Do they BUILD, SELL, or MARKET SEO/AEO capability to others?** → a competitor.

"Competitor" is broader than direct rivals: other AI SEO/AEO platforms or agencies; AI
content / writing / "marketing agent" tools positioned for ranking or answer-engine
visibility; and general AI automation / agent-builder platforms that offer or market
SEO/AEO agents or content-automation use cases, even when SEO/AEO isn't their headline.
Signals: "AI SEO", "AEO / answer-engine optimization", "GEO", "LLM visibility",
"content automation for ranking", "autonomous SEO", SEO/AEO agents.

On a conflict: **stop the paid calls**, go straight to the checkpoint with a one-line
brief recommending **drop — competitor, do not contact**, and flag it unmistakably
("COMPETITOR — do not contact") so it isn't re-prospected. **Don't over-exclude:** a
SaaS doing its own SEO is exactly who we want; when genuinely unsure, say so in the
brief and let the user call it.

## Decision aids — the four load-bearing signals

Four signals do the heavy lifting. **Assess and surface every one on every lead**, each
clearly flagged when it's missing, weak, or borderline — they are what the user's call
turns on. But they calibrate your **recommendation only**: none of them decides the
verdict, skips the checkpoint, or drops a lead on its own. Even a clear miss goes to the
user as a flag, not an auto-drop.

1. **Not a competitor.** The strongest signal and the one hard gate on *spend* (see the
   competitive-conflict gate above): a company that builds/sells/markets SEO/AEO
   capability to others is a competitor. On a conflict, recommend **drop** and stop the
   paid calls — but still route it through the checkpoint before the drop is final.
2. **Revenue in the ~$1M–$50M band.** We sell a ~$2k/mo product; below ~$1M it rarely
   pencils out, and far above the band is enterprise/procurement — the wrong motion for
   now. Slightly outside (~$900k, ~$60M) is fine — flag it. Far outside (multiples:
   $200k, $150M) is a strong drop signal. Revenue is a coarse modeled estimate — say the
   source and confidence; **unknown ARR is not a fail**, present the proxies (headcount,
   funding) instead.
3. **Real SEO motion.** An existing content/SEO footprint with real outcomes (keywords,
   organic traffic) — adopting Berlin must be a budget *reallocation*, not a cold start.
   Effectively-zero organic presence is a strong drop signal.
4. **In-house marketing capacity (strongly preferred).** A marketing/SEO team (even 1–3
   people) or a hands-on operator-founder who can run a platform. Its absence is the one
   soft-but-heavy signal — flag it clearly; don't silently drop on it.

**Everything else is soft, exploratory color** — vertical, buyer literacy, momentum, AEO
visibility, footprint size, founder-led shape. Useful in the read and worth a mention,
but never a gate: we're deliberately ranging across ICPs, so don't down-rank a lead for a
soft-signal miss. Surface it and let the user weigh it.

## Running `dogfu`

`dogfu` is a published CLI (`pip install dogfu`, or `uv tool install dogfu`). It is
**not** bundled with this skill — it talks to the Agent Berlin backend as a stateless
HTTP client.

**First step — authenticate the CLI.** Call the dogfu MCP's **`get_setup_instructions`**
tool and follow what it returns: install the `dogfu` package (if needed), then run

```bash
dogfu configure --otp <OTP> --title "Lead research: <target or batch>"
```

with the one-time OTP from that response. This is always the first step when this skill
runs.

Invoke it from anywhere: `dogfu <group> <command> [flags]`. **Output is canonical JSON
by default** — read stdout; add `-o FILE` for large payloads and read back only the
fields you need. Calls may run concurrently. The full command catalog (flags + output
fields + market codes) is in **`references/dogfu-commands.md`** — read it once up front.

The groups this skill uses: `linkedin`, `x`, `google`, `chatgpt`, `seo`, `apollo`.

## Capability → command map

### Scout

| Capability | How |
| :-- | :-- |
| Resolve company / official site | `dogfu google search --query "<name> official site"` → root domain |
| Read homepage / About / pricing / blog | **your own web-fetch tool** — derive segment, business model, audience, geography, problem statement |
| Find LinkedIn / X URLs (company or person) | `dogfu google search` / `google ai-mode` (X handles almost always come from search, not LinkedIn) |
| Company LinkedIn (headcount, funding) | `dogfu linkedin companies --url <url>` |
| Company X profile | `dogfu x profiles --url <url>` |
| Find the decision-maker | `dogfu google search --query "site:linkedin.com/in <company> (marketing OR growth OR SEO OR founder)"`, resolve with `dogfu linkedin profiles --url` |
| Page footprint | `dogfu google search --query "site:<domain>"` → `total_results` (rough; corroborate — see pipeline.md) + web-fetch `/sitemap.xml`, `/llms.txt` |
| Organic outcomes (keywords, traffic value) | `dogfu seo domain-overview --target <domain>` |
| Momentum / trend | `dogfu seo historical-rank-overview --target <domain>` |
| Firmographics (est. revenue, headcount, marketing-team size, funding, founded year) + people list | `dogfu apollo org enrich --domain <d> --with-people` (~1 Apollo credit; the people search is free) |

Capture **every person found with their LinkedIn and X URLs**, plus the **company's**
LinkedIn and X URLs — capture them whatever the user decides.

### Deep dive (after the user says pursue)

| Capability | How |
| :-- | :-- |
| Ranking quality (positions, branded share, intent) | `dogfu seo ranked-keywords --target <domain> --order-by "keyword_data.keyword_info.search_volume,desc" --limit 100` |
| Top pages by traffic (which URLs pull the traffic → DM-hook color) | `dogfu seo relevant-pages --target <domain>` |
| Technical health / stack | `dogfu seo lighthouse --url <url>` · `dogfu seo technologies --target <domain>` |
| Competitor benchmarking | discover per pipeline.md, then `dogfu seo bulk-traffic-estimation --target a.com --target b.com ...` |
| AEO / AI-answer visibility | `dogfu google ai-mode --query "..."` + `dogfu chatgpt search --query "..."` — is `<domain>` in `citations`? |
| Decision-maker deep-read | `dogfu linkedin profiles --url` + `linkedin posts --profile-url` · `dogfu x profiles --url` + `x posts --profile` (async scrapes — run detached with `-o FILE`, see the catalog) |
| Verified work email (1–2 people you'll actually contact) | `dogfu apollo people email --linkedin-url <url>` (credits per person) |
| Keyword demand / TAM (only if it changes the pitch) | `dogfu seo keyword-ideas --keyword "<seed>"` |

## The checkpoint brief — succinct, or it defeats the point

The whole reason the checkpoint exists is that the user can decide **without reading a
wall of text**. One lead:

```
**<Company>** (<domain>) — <one line: what they do, for whom, where>

| Signal | Value |
| Revenue (est.) | ~$4M (Apollo, coarse) — in band |
| SEO motion | 1.8k pages · 3.1k keywords · ~$12k/mo traffic value · rising |
| In-house team | 38 total · 2 marketing |
| Competitor? | no — uses SEO for its own growth |
| Funding | Seed, $3.5M (2024) |
| Model / market | SaaS · US · founder-led |
| Flags | none |

**Read:** <≤3 sentences: strongest signal, weakest signal, what the deep dive would settle.>
**Pursue** (deep dive: competitive gap, AEO check, top pages, contacts + hooks — the paid calls) or **drop**?
```

The first four rows are the load-bearing signals (revenue, SEO motion, in-house team,
competitor) — always present, so the user sees the whole decision at a glance.

Batch prospecting: run the Scout for all targets first, then present **one table** (one
row per lead: company · revenue · SEO motion · in-house team · competitor? · funding ·
momentum · flags · one-word read) and ask **once** which to pursue or drop. Never ask
per lead.

Rules: every number is an estimate — label anything coarse or missing rather than
implying precision. Flags column carries competitor conflicts, out-of-band size, no SEO
motion, no in-house team, unknown ARR. A recommendation is welcome; the decision is not
yours. If the session can't ask (a non-interactive run), return the brief (and the
scout data) so a human can still make the pursue/drop call.

## Optional — brand narrative coherence (only on explicit request)

*Only if the user explicitly asks* for a brand-narrative / category-coherence check,
spawn a sub-agent to execute `references/brand-narrative-coherence.md`, handing it the
domain, brand brief, geo decision, competitor shortlist, decision-maker URLs, and an
output path in the run dir; keep its substantive findings with the run's findings.
Without a clear ask, skip it.

## Operating rules

- **A sparse prompt is normal.** Derive segment, geography, audience, and competitors
  yourself; don't ask for what you can research. The checkpoint is the one mandatory ask.
- **Cheap before expensive, always**: competitor gate (free) → SEO motion (`site:`,
  `domain-overview`) → the rest of the Scout → checkpoint → deep dive. Stop the paid
  calls the moment the gate says competitor.
- **Never decide for the user.** No deep dive without a pursue; no verdict the user
  didn't give. This holds even when the data is clear-cut — recommend, don't decide.
- **Never waste research.** Capture everything found — attributes, contacts, links,
  findings, the user's decision + reason — for a drop as much as a pursue, and keep the
  raw pulls in the run dir so nothing needs re-fetching.
- **Scratch files go in one `mktemp -d` run dir** — never a fixed or target-derived
  path (parallel batch runs clobber shared paths).
- **Numbers are modeled estimates.** Use them comparatively; flag implausible ones.
- **Specify the market.** SEO data is market-specific — set `--location-code` /
  `--language-code` from the derived geography; a global SaaS may warrant two markets.
