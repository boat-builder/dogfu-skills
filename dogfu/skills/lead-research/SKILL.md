---
name: lead-research
description: >-
  Run Agent Berlin's staged lead-research pipeline — Scout → checkpoint → Deep dive →
  CRM — using the `dogfu` CLI. Use this whenever the user hands you a sales target (a
  company name, a domain, a person, or a LinkedIn/X URL) and wants it researched, sized
  up, enriched for outreach, or pushed into the Close CRM. Triggers include "research
  this lead", "qualify this prospect", "is this company a fit", "add this company to the
  CRM", "find the decision-maker and DM hooks", "prospect this domain", or just a bare
  LinkedIn/X/company link dropped in with intent to evaluate it. Also use it for batch
  prospecting. The skill gathers cheap signals first, presents a succinct decision
  brief, and the USER decides whether to spend on the deep dive and what status the lead
  gets — the skill never marks a lead Qualified or Bad Fit on its own. Every researched
  lead lands in the CRM with structured company attributes. This is the canonical way to
  do sales lead research at Berlin — prefer it over ad-hoc web searches.
---

# Lead Research (powered by `dogfu`)

Turn a sparse sales **target** into a researched CRM record and a **human decision**.
You gather the cheap, high-signal facts first, compress them into a short decision
brief, and the user decides whether the lead is worth the expensive deep dive — and
what status it gets. You research and recommend; **you never decide**.

This skill is the **orchestration layer**. The data and CRM actions come from the
`dogfu` CLI (CRM writes go via the Close key connected in the Console — see "Running
`dogfu`" below). Your job is to run the right `dogfu` commands in the right order, read
the JSON they return, and reason over it. The stage-by-stage interpretation logic lives
in **`references/pipeline.md`** — read it before running the Scout.

## The flow

1. **Scout** *(always)* — resolve the target to a company + root domain, read its own
   pages and socials, find the decision-maker(s), then pull the **cheap** SEO and
   firmographic signals. Capture every company attribute you find.
2. **Checkpoint** *(always)* — present the decision brief (format below) and ask the
   user: **pursue / park / drop**. This is the only place spend escalates and the only
   place a status is chosen. Never skip it, never answer it yourself.
3. **Deep dive** *(pursue only)* — ranking quality, technical health, competitive gap,
   AEO visibility, decision-maker deep-reads, verified emails, DM hooks.
4. **CRM write** *(always, at whatever stage the run ends)* — upsert the lead with a
   brief description and **every curated attribute flag you can fill**, add every person
   found as a contact with their links, write the research note. The user's checkpoint
   decision sets the status.

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
brief recommending **drop — competitor, do not contact**, and after the user confirms,
record it (status Bad Fit; description and note say "COMPETITOR — do not contact") so
it is never re-prospected. **Don't over-exclude:** a SaaS doing its own SEO is exactly
who we want; when genuinely unsure, say so in the brief and let the user call it.

## Decision aids — what the brief measures against

These are the signals we currently know matter. They calibrate your **recommendation**;
they never auto-set a status or skip the checkpoint.

- **ARR band ~$1M–$50M.** Slightly outside (~$900k, ~$60M) is fine — flag it. Far
  outside (multiples: $200k, $150M) is a strong drop signal. Revenue numbers are coarse
  modeled estimates — say the source and confidence, and **unknown ARR is not a fail**;
  present the proxies (headcount, funding) instead.
- **Real SEO motion is mandatory.** An existing content/SEO footprint with real
  outcomes (keywords, organic traffic) — adopting Berlin must be a budget reallocation,
  not a cold start. Effectively-zero organic presence is a strong drop signal.
- **In-house marketing capacity is preferred.** A marketing/SEO team (even 1–3 people)
  or a hands-on operator-founder who can run a platform.
- **Known-good shape:** founder-led SaaS doing its own SEO. Worth saying in the brief
  when it matches; not a gate.

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

The groups: `linkedin`, `x`, `google`, `chatgpt`, `seo`, `crm`, `apollo`.

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
LinkedIn and X URLs — they persist to the CRM whatever the user decides.

### Deep dive (after the user says pursue)

| Capability | How |
| :-- | :-- |
| Ranking quality (positions, branded share, intent) | `dogfu seo ranked-keywords --target <domain> --order-by "keyword_data.keyword_info.search_volume,desc" --limit 100` |
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
| ARR (est.) | ~$4M (Apollo, coarse) — in band |
| Team | 38 total · 2 marketing |
| Funding | Seed, $3.5M (2024) |
| SEO motion | 1.8k pages · 3.1k keywords · ~$12k/mo traffic value · rising |
| Model / market | SaaS · US · founder-led |
| Flags | none |

**Read:** <≤3 sentences: strongest signal, weakest signal, what the deep dive would settle.>
**Pursue** (deep dive: competitive gap, AEO check, contacts + hooks — the paid calls), **park**, or **drop**?
```

Batch prospecting: run the Scout for all targets first, then present **one table** (one
row per lead: company · ARR · team/mktg · funding · SEO motion · momentum · flags ·
one-word read) and ask **once** which to pursue, park, or drop. Never ask per lead.

Rules: every number is an estimate — label anything coarse or missing rather than
implying precision. Flags column carries competitor conflicts, out-of-band size, no SEO
motion, unknown ARR. A recommendation is welcome; the decision is not yours. If the
session can't ask (non-interactive run), record everything as **Potential** with the
brief in the note and a Close task "Review for qualification" so the ask surfaces in
the worklist.

## CRM write — always, fields in their proper place

Runs for every lead at whatever stage the run ended. Upsert on the resolved domain
(search before create). If Close isn't connected, `dogfu crm` returns a `412` — surface
it, don't retry.

**Status — set from the user's checkpoint decision, never from your own judgment:**

| Decision | Status | Note |
| :-- | :-- | :-- |
| pursue | **Qualified** | enters the outreach flow (the reach-out task opens on this status) |
| park / undecided / non-interactive | **Potential** | researched, decision deferred |
| drop | **Bad Fit** | record the user's reason; competitors additionally get "COMPETITOR — do not contact" in the description |

**Where each piece of data goes** (statuses: `crm status list` — never hardcode ids):

| What | Where | Command |
| :-- | :-- | :-- |
| Duplicate check | — | `dogfu crm lead list -n "<company>"` (or `-q "<domain>"`) before creating |
| Name, website, **brief** 1–2 line description, status | native lead fields | `dogfu crm lead create -n .. -u .. -d .. -s ..` / `lead update` |
| **Company attributes — fill every flag you have a value for; skip the rest** | curated lead flags | `--industry` `--business-model` `--primary-market` `--year-founded` `--employees` `--marketing-team-size` `--revenue` `--funding-stage` `--total-funding` `--seo-pages` `--organic-keywords` `--seo-investment-tier` `--seo-momentum` `--aeo-visibility` `--company-linkedin` `--company-x` |
| Each person found (fit or not) | contact + native `urls` | `dogfu crm contact create <lead_id> -n .. -t .. -u <linkedin> -u <x> [-e <verified email>] [-p <phone>]` |
| The research depth: what they do, market, metrics with sources, the brief's read, the user's decision + reason, and (pursued) DM hooks + competitive gap | lead note | `dogfu crm note create <lead_id> -t "<write-up>"` |

The attribute → source mapping (which Scout/deep-dive datum fills which flag) is in
`references/pipeline.md`. Choices fields (`--industry`, `--funding-stage`, …) validate
against Close's live options — on a rejection the error lists the allowed values; pick
the closest or omit and note it. `dogfu crm field get "<name>"` shows a field's options;
`crm field add-choice`/`remove-choice` edit them, but only on the user's explicit ask.

Running this skill is the go-ahead for these CRM writes — show what you're writing in
the run summary, don't pause per write. The **checkpoint** is the only pause.

## Output format (end of run)

After a **drop/park**: the one-line outcome + the CRM record written. After a
**pursue** (deep dive done):

1. **Snapshot** — target, domain, segment, decision, in 2–3 sentences.
2. **Profile table** — each signal: headline number + one-line read.
3. **Competitive gap** — target vs leader/median, as ratios.
4. **Contacts & links** — decision-maker(s) with LinkedIn/X URLs, verified emails, and
   ready-to-use DM hooks, plus a **touch-0 starter**: the profile(s) to send a
   connection request to and 1–3 specific recent posts (with URLs) to like or comment
   on. The BDR acts by hand and records it via the lead-touch skill.
5. **Key gaps & angles** — divergences framed as outreach openings.
6. **Evidence appendix** — the exact `dogfu` commands run, so the run is reproducible.

## Optional — brand narrative coherence (only on explicit request)

*Only if the user explicitly asks* for a brand-narrative / category-coherence check,
spawn a sub-agent to execute `references/brand-narrative-coherence.md`, handing it the
domain, brand brief, geo decision, competitor shortlist, decision-maker URLs, and an
output path in the run dir; persist its substantive findings to the CRM note. Without a
clear ask, skip it.

## Operating rules

- **A sparse prompt is normal.** Derive segment, geography, audience, and competitors
  yourself; don't ask for what you can research. The checkpoint is the one mandatory ask.
- **Cheap before expensive, always**: competitor gate (free) → SEO motion (`site:`,
  `domain-overview`) → the rest of the Scout → checkpoint → deep dive. Stop the paid
  calls the moment the gate says competitor.
- **Never decide for the user.** No status without a checkpoint decision; no deep dive
  without a pursue.
- **Never waste research.** Everything found gets written to the CRM — contacts, links,
  attributes, findings — whatever the decision was.
- **Put data in its proper place.** Brief description; attributes in their flags;
  person links on the contact; depth in the note.
- **Scratch files go in one `mktemp -d` run dir** — never a fixed or target-derived
  path (parallel batch runs clobber shared paths).
- **Numbers are modeled estimates.** Use them comparatively; flag implausible ones.
- **Specify the market.** SEO data is market-specific — set `--location-code` /
  `--language-code` from the derived geography; a global SaaS may warrant two markets.
