---
name: narrative-audit
description: >-
  Run a **brand narrative & category-coherence** check on a domain — do the market, the brand's
  own site, its customers, third-party sources, the founder, and live AI answers all tell the
  **same story about what category the brand is in and why it belongs**? Use when someone asks to
  "run a narrative coherence check", "audit our brand story", "is our positioning consistent
  across sources", "do our site, G2, and ChatGPT say the same thing", "category coherence audit",
  or "check narrative coherence for <domain>". Outside-in, from public data via the `dogfu` CLI +
  native web reading; the research and analysis run in a sub-agent. For a full SEO/AEO prospect
  audit use first-audit; to research/qualify a brand-new target use lead-research.
---

# Narrative Audit — brand category & narrative coherence

AI assistants recommend a brand when every source they read agrees on **what category it's in
and why it belongs**. This skill measures that coherence for one domain and returns where the
sources agree, where they diverge, and **why each divergence costs AI selection** — as a
standalone deliverable.

The heavy research + analysis runs in a **sub-agent**, so this skill's own context stays lean —
you hand off the playbook and inputs, and present what comes back.

## What you need first
- **Authenticate the `dogfu` CLI.** Call the dogfu MCP's **`get_setup_instructions`** and follow
  it (install `dogfu` if needed, then `dogfu configure --otp <OTP> --title "Narrative audit:
  <domain>"`). Stateless HTTP client; canonical JSON by default; `-o FILE` to dump large payloads.
- **Native web reading** for pages `dogfu` doesn't model (review sites, press, the brand site).
- A **run dir** via `mktemp -d` — everything this run writes goes inside it.

## How it runs
1. **Resolve the domain** from the request. *Optional seed:* if the brand is already in Close,
   pull its lead-research profile as a **reference prior** (`dogfu crm lead list -q <domain>` →
   `lead get` / `note list`, read-only) — a seed for the brand brief, geo, and competitors, never
   a substitute for fresh reads. Skip on `412` / no match.
2. **Spawn the coherence sub-agent.** Hand it:
   - the playbook — **`references/brand-narrative-coherence.md`** (in this skill's directory); it
     reads and executes this;
   - the inputs the playbook lists (domain; plus any brand brief / geo / competitors / AI-answer
     citations / founder LinkedIn/X you already have — so it doesn't re-fetch);
   - an **output path** in the run dir (e.g. `<run-dir>/narrative-coherence.md`).

   The sub-agent does all discovery, reading, and the coherence judgment, and writes the file.
3. **Deliver.** Read the sub-agent's output and present it: the coherence score, the
   category-placement matrix, the divergence map, the substantiation gaps, and the panel report.
   Every finding must carry its **evidence + reasoning** — never a bare score.

## Guardrails
- **The sub-agent owns the playbook.** You don't read `references/brand-narrative-coherence.md`
  yourself — hand its path off and keep your context clean.
- **Outside-in only.** Public data; you don't own this domain. No CRM writes unless asked.
- **Cost discipline** (carried in the playbook): cheap SERP/AI probes first, native reads only
  where a snippet won't settle a claim, founder pulls skip-if-silent.
- **One run dir** (`mktemp -d`) holds every scratch file and the output.
