---
name: lead-research
description: >-
  Run Agent Berlin's full lead-research pipeline — Discover → Qualify → Enrich → CRM —
  using the `dogfu` CLI. Use this whenever the user hands you a sales target (a company
  name, a domain, a person, or a LinkedIn/X URL) and wants it researched, qualified
  against an ICP, enriched for outreach, or pushed into the Close CRM. Triggers include
  "research this lead", "qualify this prospect", "is this company a fit / in our ICP",
  "add this company to the CRM", "find the decision-maker and DM hooks", "prospect this
  domain", or just a bare LinkedIn/X/company link dropped in with intent to evaluate it.
  Also use it for batch prospecting and for recording disqualified leads. The skill
  ships its ICPs in `icps/` and qualifies against the default unless the caller names a
  specific one or supplies their own. This is the canonical way to do sales lead research
  at Berlin — prefer it over ad-hoc web searches.
---

# Lead Research (powered by `dogfu`)

Turn a sparse sales **target** into a qualified, enriched CRM record. You discover the
company and the right people, qualify them against an ICP, and — if they fit — enrich
them with the raw material for a personalized DM, then persist **everything you found** to
the CRM (whether or not they fit).

This skill is the **orchestration layer**. The data and CRM actions come from the
`dogfu` CLI, authenticated through the dogfu MCP (data calls via your session token; CRM via
the Close key you connect in the Console — see "Running `dogfu`" below). Your job is to run
the right `dogfu` commands in the right order, read the JSON they return, and reason over it.

## The four stages

1. **Discover** — resolve the target to a company + root domain, read the company, map its social footprint, find the marketing/SEO decision-maker(s) and their profile links. *Always runs.*
2. **Qualify** — judge ICP fit using the runtime ICP, the discovery findings, and an SEO/AEO profile you build from `dogfu seo` data. *Always runs.* It opens with a **competitive-conflict gate** — exclude companies that build or sell SEO/AEO capability (even adjacent AI/automation/agent platforms), since they're competitors, not customers. Stop the deep, expensive work the moment cheap signals clearly disqualify.
3. **Enrich** *(fits only)* — pull **Apollo** firmographics (estimated revenue, headcount, marketing-team size, funding) and the decision-maker's **verified work email**, deep-read the decision-maker(s), and pull DM hooks. *Only runs if the verdict is a fit (strong or partial)* — these are the expensive/credit-bearing pulls, so it's the one stage gated on fit.
4. **CRM write** — persist everything gathered — qualified or not — so a "no" is recorded, the research isn't wasted, and you keep the contact links needed to reach out later. *Always runs.*

Be **cost-aware throughout**: every `dogfu` SEO/social call hits a paid backend. Pull
cheap, high-signal data first; only go deep once the target looks plausibly in-ICP. But
once you've spent the money to find something, **always save it to the CRM** (Stage D) —
even for non-fits.

The detailed stage-by-stage profiling logic (the qualification phases, the synthesis
tiers, the divergence checks, the calibration rules) lives in **`references/pipeline.md`**.
Read it before running Stage B — it is the substance of the qualification. This SKILL.md
tells you *which `dogfu` command fulfils each step*; `pipeline.md` tells you *how to
interpret the result*.

***

## The ICP — bundled in `icps/`, with a default

This skill **ships its ICPs** in the `icps/` folder — one markdown file per ICP. Each ICP
file has frontmatter (`name`, a human `label`, and `default: true` on exactly one) and a
body that is your qualification yardstick — judge against it and map each phase's evidence
back to its specific criteria. (Files named `README.md` or starting with `_` are
scaffolding, not ICPs — ignore them.)

**Choose which ICP to qualify against, in this order:**

1. **Caller override** — if the caller pasted an ICP inline or gave a file path, use that.
2. **Named ICP** — if the caller named one (e.g. "use the enterprise ICP"), match it to an
   `icps/*.md` file by its `name`/`label` and use that file.
3. **Default** — otherwise use the file marked `default: true` (or the only ICP, if there
   is just one).

Read the chosen file and treat its body as the ICP for this run. **Only ask the caller for
an ICP** if `icps/` has none and the caller supplied none — never invent one. When several
ICPs exist but none is marked `default` and the caller didn't name one, **ask which**.
(Stage A discovery can run meanwhile; it doesn't depend on the ICP.)

***

## Competitive-conflict gate — exclude competitors before qualifying

Berlin is an **AI SEO/AEO platform** that replaces hiring an SEO agency or in-house team.
We don't pitch companies that compete with that. So before spending on qualification, apply
one test to what Stage A told you about *what the company does*:

- **Do they USE SEO/AEO for their own growth?** → that's the ICP. Keep going.
- **Do they BUILD, SELL, or MARKET SEO/AEO capability to others?** → that's a competitor. **Exclude them.**

"Competitor" is broader than direct rivals — catch the **adjacent** cases too:
- Other AI SEO/AEO platforms, tools, or agencies productizing AI for SEO/AEO/GEO.
- AI content / writing / "marketing agent" tools positioned for ranking or answer-engine visibility.
- **General AI automation / agent-builder platforms that offer or market SEO/AEO agents or content-automation use cases** — even when SEO/AEO isn't their headline. If their platform produces SEO/AEO outcomes for *their* users, it overlaps with what Berlin sells.

Signals on their site / product / pricing / marketing: they offer users things like "AI SEO",
"AEO / answer-engine optimization", "GEO", "LLM visibility", "content automation at scale for
ranking", "autonomous SEO", rank-tracking-plus-action, or SEO/AEO agents.

**Run this first — it's the cheapest disqualifier**, right after Stage A tells you what the
company does and before any paid SEO call. If it's a conflict, the verdict is **excluded
(competitor)** regardless of how perfect the behavioral fit looks; skip the rest of Stage B
and Stage C, but still write the CRM record (Stage D) so they aren't re-prospected.

**Don't over-exclude.** Most ICP targets are SaaS companies that legitimately do their own
SEO — that's exactly who we want. Only exclude when they provide SEO/AEO capability *to
others*. When genuinely unsure, flag the ambiguity for a human rather than auto-excluding a
strong behavioral fit.

***

## Running `dogfu`

`dogfu` is a published CLI (`pip install dogfu`, or `uv tool install dogfu`), authenticated
through the **dogfu MCP**. It is **not** bundled with this skill — there is no directory to
mount and no `.env`. The CLI talks to the Agent Berlin backend over an authenticated session
token.

**Setup (usually already done when the dogfu MCP is connected).** If `dogfu` isn't installed
or configured yet — or a command fails with an auth error (`missing dogfu token`, `invalid or
expired token`) — call the dogfu MCP's **`get_setup_instructions`** tool and follow what it
returns: install the `dogfu` package, then run

```bash
dogfu configure --otp <OTP> --title "Lead research: <target or batch>"
```

with the one-time OTP from that response. The token is saved to `~/.dogfu/config.json`. After
an auth failure, call `get_setup_instructions` again for a fresh OTP, re-run `dogfu
configure`, and retry the command.

**Invoke it from anywhere** (it's on your PATH after install — no `uv run`, no mounted
directory):

```bash
dogfu <group> <command> [flags]
```

**Output is canonical JSON by default** — pipe nothing, just read stdout. Add `-o FILE`
to save a large payload to a file instead of flooding your context, then read back only
the fields you need (see the "unique run dir" rule below). `-f table` is for humans, not for you.

You can run calls concurrently if it helps — the CLI is a stateless HTTP client with no
single-process lock. **Cost discipline still applies:** prefer the cheapest informative call
next rather than fanning out paid calls speculatively.

**Discovery:** `dogfu --help`, `dogfu <group> --help`, and `dogfu <group> <cmd> --help`
print exact flags and the `Output:` shape. A full command catalog with output fields is
in **`references/dogfu-commands.md`** — read it once up front so you don't burn calls on
`--help`.

The groups: `linkedin`, `x`, `google`, `chatgpt`, `seo`, `crm`, `apollo`.

***

## Capability → `dogfu` command map

The pipeline is written in terms of *capabilities*. Here is how each maps to a concrete
command. Where `dogfu` has no command for a capability, use your own tools (web fetch,
the WebSearch tool) — those cases are called out explicitly.

### Discover (Stage A)

| Capability | How |
| :-- | :-- |
| Resolve company / find official site | `dogfu google search --query "<name> official site"` → take the root domain |
| Read homepage / About / pricing / blog | **Your own web-fetch tool** (`dogfu` has no page-fetch). This is where you derive segment, audience, geography, problem statement. |
| **Find the LinkedIn / X URLs (company or person) when you don't have them** | Use `dogfu google search` or `dogfu google ai-mode` — e.g. `--query "<company> LinkedIn"`, `--query "<company> X (Twitter)"`, `--query "<person name> <company> LinkedIn"`, `--query "<person name> X / Twitter handle"`. LinkedIn doesn't expose a person's X handle, so X handles in particular almost always come from a Google / AI-mode search. Take the canonical profile URL from the results, then hydrate it with the dedicated command below. |
| Company LinkedIn page | `dogfu linkedin companies --url <linkedin-company-url>` → description, headcount, industries, funding |
| Find a person's profile by name | `dogfu linkedin profiles --name "Full Name"` (discovery) or `--url <profile-url>`. **Profiles can come back sparse** — `experience[]` and even `current_title` are sometimes empty (private profiles). Corroborate background from their posts and a web search rather than trusting the profile call alone. |
| Company X/Twitter profile | `dogfu x profiles --url <x-url>` (find the handle first with `dogfu google search` / `ai-mode` per the row above) |
| Find a person's X handle | `dogfu google search` / `dogfu google ai-mode` (LinkedIn won't give it) — then `dogfu x profiles --url` |
| Map employees / find the decision-maker | `dogfu linkedin companies` gives headcount but **not a full employee list**. To find the marketing/SEO owner, `dogfu google search --query "site:linkedin.com/in <company> (marketing OR growth OR SEO OR founder)"`, then resolve names with `dogfu linkedin profiles --name`. |

Capture **every person you find with their LinkedIn and X URLs**, plus the **company's**
LinkedIn and X URLs — these are the outreach handles you'll persist in Stage D, fit or not.

### Qualify (Stage B) — SEO/AEO data

| Capability | How |
| :-- | :-- |
| Page footprint / scale | `dogfu google search --query "site:<domain>"` → read `total_results`. **Caveat:** a bare `site:rootdomain` is a rough Google estimate often dominated by subdomains and can understate the real content footprint ~10×. Corroborate with the keyword count from `domain-overview` and a narrower `site:www.<domain>/blog` query; treat the numbers as a range. Plus web-fetch `<domain>/sitemap.xml` and `<domain>/llms.txt` (presence is a signal). |
| Top pages by traffic | `dogfu seo relevant-pages --target <domain>` |
| Domain organic overview (traffic, traffic value, # keywords) | `dogfu seo domain-overview --target <domain>` — returns organic `count` (≈ keyword count) and `etv` (traffic value). |
| Momentum / trend over time | `dogfu seo historical-rank-overview --target <domain>` |
| Ranking quality (positions, branded vs not, intent) | `dogfu seo ranked-keywords --target <domain> --order-by "keyword_data.keyword_info.search_volume,desc" --limit 100` |
| Keyword demand / TAM | `dogfu seo keyword-ideas --keyword "<seed>"` |
| Technical health (Core Web Vitals) | `dogfu seo lighthouse --url <url>` |
| Detected tech stack + domain strength hint | `dogfu seo technologies --target <domain>` — also returns `domain_rank`, a **large integer (DataForSEO-style rank), not a 0–100 score**; its scale/direction is ambiguous, so use it only as a coarse, *relative* signal (target vs competitors), never as an absolute grade. |
| AI-answer visibility & citations | `dogfu google ai-mode --query "..."` and `dogfu chatgpt search --query "..."` — check whether `<domain>` appears in `citations`. |
| Competitor benchmarking (footprint/traffic for many domains at once) | `dogfu seo bulk-traffic-estimation --target a.com --target b.com ...` |

### Enrich (Stage C) — fit only

**Runs only for strong/partial fits.** The Apollo calls cost credits, so the ICP verdict
gates them — never pull Apollo to qualify. They need the Apollo key connected in the
Console → **Apollo Integration** (a `412` "no Apollo API key configured" means it isn't —
surface that, like the CRM key).

| Capability | How |
| :-- | :-- |
| **Firmographics + decision-makers (Apollo)** | `dogfu apollo org enrich --domain <d> --with-people [--title "Head of Marketing" --title "SEO Manager" ...]` → est. `annual_revenue`, `employee_count`, `marketing_headcount`, funding, tech, and `people[]` (name, title, seniority, `linkedin_url` — **no email**). Firms up the ICP size/revenue read (the band the SEO data can't measure) and surfaces who to contact. ~1 Apollo credit; the people search itself is free (needs a master key — without one the org still returns, `people[]` empty). |
| **Verified work email (Apollo)** | `dogfu apollo people email --linkedin-url <url>` (best) or `--name "<full name>" --domain <d>`. Take the person from the `org enrich --with-people` list (or Stage A discovery). Returns the verified work `email` + `email_status`. Consumes credits — resolve only the **1–2 decision-makers you'll actually contact**, not everyone. |
| Deep-read decision-maker on LinkedIn | `dogfu linkedin profiles --url <profile>` (background, experience) + `dogfu linkedin posts --profile-url <profile>` (recent posts) |
| Deep-read decision-maker on X | `dogfu x profiles --url <profile>` + `dogfu x posts --profile <profile>` |

### CRM write (Stage D) — Close

CRM writes go through the backend proxy under the caller's **own** Close key (set in the
Console → CRM Integration; see "Running `dogfu`"). If Close isn't connected, `dogfu crm`
commands return a "no Close CRM API key configured" / `412` error — surface that to the user
rather than retrying. Running this skill **is** the go-ahead for its Stage D writes: show the
planned lead/contact/note in your run summary, but don't pause for per-write confirmation (it
would break batch prospecting).

Put each piece of data **where it belongs in the Close UI**, not all in one blob:

| What | Where it goes | Command |
| :-- | :-- | :-- |
| Status IDs (account-specific — never hardcode) | — | `dogfu crm status list` |
| Duplicate check (upsert) | — | `dogfu crm lead search --name "<company>"` (or `--query "<domain>"`) **before** creating |
| Company name + website + **brief** summary + status | native lead fields | `dogfu crm lead create -n "<company>" -u <domain> -d "<1–2 line summary>" -s <status_id>` / `lead update <lead_id> ...` |
| Curated company facts (the *only* custom fields to set) | dedicated lead flags | the five `--industry / --employees / --revenue / --business-model / --seo-pages` flags below |
| **A person's LinkedIn / X links** | **native contact `urls` field** (renders on the contact card in Close) | `dogfu crm contact create <lead_id> -n "<name>" -t "<title>" -u <linkedin-url> -u <x-url>` — `-u` is repeatable; also pass `-e <email>` (the Apollo verified work email, for fits — it unlocks the email outreach channel) / `-p <phone>` when found |
| Everything else — full profile, metrics, momentum, competitive gap, verdict, **company LinkedIn/X links**, contact location/seniority, DM hooks | the lead **Note** ("Notes & summaries") | `dogfu crm note create <lead_id> -t "<the structured write-up>"` |

#### Curated lead custom fields — set these, and only these

`dogfu` exposes a **specific, curated** set of company custom fields as named flags on
`crm lead create` / `lead update`. Set each one **when you have the value** (skip the flag
when you don't — never pass a blank); **anything that isn't one of these goes in the
description or the note.** Don't try to set other custom fields — there are no flags for
them by design.

| Flag | Fill from | Notes |
| :-- | :-- | :-- |
| `--employees <n>` | headcount — LinkedIn `company.employee_count`, or Apollo `employee_count` (fits) | a number |
| `--revenue <usd>` | annual revenue | a number in USD. **For fits, from Apollo `annual_revenue`** (a modeled estimate — coarse for small private SaaS, so flag it as such); otherwise usually unknown — then skip it and leave the size read in the note. |
| `--business-model "<...>"` | derived business model (A2) | free text, e.g. `SaaS`, `Marketplace`, `Services`, `Dev tool` |
| `--industry "<choice>"` | derived vertical, mapped to Close's **allowed choices** | validated server-side. Map your sector to the closest choice (e.g. SaaS → `Software`). If the value is rejected, `dogfu` returns the allowed list — retry with the closest, or omit and note it. Don't block the run on this. |
| `--seo-pages <n>` | indexed-page footprint from B1 (`site:` estimate) | a number |

Example: `dogfu crm lead create -n "Acme" -u acme.com -s <id> -d "<brief>" --business-model SaaS --industry Software --employees 140 --seo-pages 1800`

**Keep the lead `description` brief** — one or two lines, like a headline (segment +
verdict + the single lead-with angle). The curated facts go in their flags; all remaining
depth goes in the **note**, so the Close lead view stays scannable.

**Links land in proper fields:** a person's LinkedIn/X go in that contact's native `urls`
field (so you can message them straight from Close), not buried in prose. Company-level
LinkedIn/X have no dedicated flag, so include them in the note.

**Status mapping** (resolve the live IDs with `crm status list` — labels may differ):
fit → **Qualified**; not in ICP → **Bad Fit**; unsure / not yet worked → **Potential**.

***

## Workflow

1. **Pick the ICP.** Use the caller's ICP if they pasted one or named a specific `icps/*.md`; otherwise read the bundled default (see "The ICP" above). Only ask if `icps/` is empty and none was supplied (Stage A can run meanwhile).
2. **Stage A — Discover.** Resolve the target → domain. Web-fetch the company's own pages to derive segment, audience, geography/language, and the problem statement (you need these to calibrate every SEO threshold and to discover competitors). Find and record the company's LinkedIn + X URLs and each decision-maker's LinkedIn + X URLs (use Google search / AI-mode to locate any you weren't handed) — you persist these regardless of fit.
3. **Stage B — Qualify.** **First, apply the competitive-conflict gate** (above) using what Stage A told you about the business — if the target builds/sells/markets SEO/AEO capability (directly or via a broader AI/automation/agent platform), stop and mark **excluded (competitor)**, skipping the paid SEO calls. Otherwise read `references/pipeline.md`, then run the phases with the commands above, **cheap signals first**. Set the SEO data's market with `--location-code` / `--language-code` (see the codes in `references/dogfu-commands.md`) from the geography you derived. Calibrate to the segment and to discovered competitors (ratios, not absolute numbers). Stop early and mark "not in ICP" if cheap signals clearly disqualify. Produce a verdict: **strong / partial / weak**.
4. **Stage C — Enrich** (only if strong/partial — this is where the credit-bearing Apollo pulls live). Run `dogfu apollo org enrich --domain <d> --with-people` to firm up the size/revenue read (est. revenue, headcount, marketing-team size, funding) and surface the decision-makers; then `dogfu apollo people email` (by `--linkedin-url`, else `--name --domain`) for the verified work email of the **1–2 people you'll actually contact**. Deep-read each decision-maker (LinkedIn + X profile and recent posts). Pull personal hooks, company hooks (the SEO/AEO gaps from Stage B, framed as openings), and the relevance bridge to Berlin.
5. **Stage D — CRM write (always, fields in their proper place).** Upsert the lead on its domain with a **brief** description, and set the curated lead flags (`--employees`, `--business-model`, `--industry`, `--seo-pages`, `--revenue`) for whatever you found — skip any you don't have. Add **every person you found as a contact, with their LinkedIn/X in the native `urls` field**. Write a note with **all the research** — what they do, who they serve, company LinkedIn/X links, SEO/AEO metrics, verdict + reason, and (for fits) the DM hooks. **Do this for non-fits too:** mark them Bad Fit with the reason, but still save the contacts, their links, and everything you found — the research cost real money and effort, so none of it should be discarded. Set status from the verdict.
6. **Return the run summary** in the output format below.

## Output format

In addition to the CRM write, return:

1. **Snapshot** — target, resolved domain, segment, market, decision-maker(s), ICP-fit verdict, in 2–3 sentences.
2. **Profile table** — the Stage B dimension table, each row a headline number + one-line read.
3. **Competitive gap** — target vs leader/median as ratios.
4. **Contacts & links** — decision-maker(s) with their LinkedIn/X URLs and (if a fit) ready-to-use DM hooks. **For a qualified lead, also include a "touch 0" starter the BDR can act on immediately: the profile(s) to send a connection request to, and 1–3 specific recent posts (with their URLs, from the Stage C reads) to like or comment on. Touch 0 is one or more of just those three actions — connection request, like, or comment; the BDR does it by hand, then records it via the lead-touch skill.**
5. **Key gaps & angles** — the divergences found, framed as outreach openings.
6. **Evidence appendix** — the exact `dogfu` commands run and key fields pulled, so the run is reproducible. Keep all traffic/keyword numbers labeled as estimates.

## Operating rules

- **Treat a sparse prompt as normal.** Derive segment, geography, audience, and competitors yourself before qualifying; don't stop to ask for things you can research (the ICP comes from `icps/` — only ask if none is bundled and none was supplied).
- **Exclude competitors.** Before paid qualification, run the competitive-conflict gate: if the target builds/sells/markets SEO/AEO capability to others (incl. adjacent AI automation/agent platforms), mark it excluded (competitor) and don't reach out — but still record it so it isn't re-prospected.
- **Cost discipline.** Cheap, high-signal first (`site:` count, `domain-overview`); go deep (`ranked-keywords`, `lighthouse`, `bulk-traffic-estimation` on competitors, deep social reads) only once the target looks plausibly in-ICP.
- **Apollo is fit-gated.** `apollo org enrich` and `apollo people email` consume Apollo credits — run them **only for strong/partial fits** (Stage C), never to qualify. They supply the revenue/headcount the SEO data can't and the decision-maker's verified work email; resolve emails only for the 1–2 people you'll actually contact.
- **Never waste research.** Whatever you spent money to discover gets written to the CRM in Stage D — contacts, links, metrics, and findings — fit or not.
- **Put data in its proper place.** Brief lead description; person links in the contact `urls` field; everything else in the note.
- **Write scratch files to a unique run dir.** Anything this run saves to disk — `dogfu -o` dumps, research/brand-profile notes, any future scratch file — goes inside **one** directory made with `mktemp -d` at the start; never a fixed or target-derived path. Parallel runs (batch prospecting, concurrent calls) clobber shared paths.
- **Numbers are modeled estimates.** Use them comparatively (target vs competitor vs segment); say so when one looks implausible rather than reporting it straight.
- **Specify the market.** SEO data is market-specific. A global SaaS may warrant checking more than one market.
- **The CRM write always happens** — keyed on the domain to avoid duplicates.
