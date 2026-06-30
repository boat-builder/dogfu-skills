# lead-engage — skill source

This is the `lead-engage` skill, shipped inside the **`dogfu`** plugin. It's the **warm-phase**
counterpart to `lead-touch`: where `lead-touch` runs the *cold* outreach cadence and hands a
lead off the moment it replies (→ **Engaged**), `lead-engage` works the live deal from there —
discovery, an **opportunity**, a trial, a proposal, won or lost. It triggers when someone wants
to act on a lead that has already engaged: run/relay a discovery call, open or advance or close
an opportunity, log a call/demo/proposal, set or check a deal's next step, pull the deals due or
stalled, mark a deal won/lost, intake an inbound lead, or report the deal pipeline / forecast.

It's the third phase of the lifecycle:

```
lead-research  →  lead-touch        →  lead-engage
(Discover &       (cold cadence;        (live deal; Engaged → Won/Customer or lost)
 Qualify)          reply → Engaged)
```

## Files
- `SKILL.md` — the operating manual: the two-layer model (lead **status** = account lifecycle
  vs **opportunity** = the forecastable deal), the **qualification gate** (open an opportunity
  only after a call confirms a real deal — never on a reply), the lean pipeline (Discovery →
  Trial → Proposal → Won/Lost), the **next-step**-driven work queue (due / dropped-ball /
  stalled), inbound intake, the `dogfu crm` command surface incl. the new `opportunity` verbs,
  and the operating rules. The YAML frontmatter `name` / `description` controls when the skill
  triggers.

(The opportunity build spec and its follow-up deltas live with the CLI devs, not in this repo —
see the dependency note below.)

## The `dogfu` CLI dependency
Same as the other dogfu skills: the skill is the orchestration layer and shells out to the
published `dogfu` CLI (`pip install dogfu`) for all data and CRM actions, authenticated through
the **dogfu MCP** (`get_setup_instructions` → `dogfu configure --otp <OTP> --title "…"`). CRM
(Close) calls go through the backend under the Close API key set once in **Console → CRM
Integration**.

## Status — CLI shipped; pending the Close opportunity pipeline
The **`dogfu crm opportunity …`** commands have shipped and the **`Engaged`** lead status is live
(with `Trial` removed). What remains before the skill is fully operational is the Close
**opportunity pipeline** (Discovery → Trial → Proposal → Won/Lost) and the **`Deal Type`** custom
field. Until those exist, the skill can read/move lead **statuses** but can't yet work
opportunities. The full build spec and its follow-up deltas live with the CLI devs, not in this repo.

## Why a separate skill (not a mode of `lead-touch`)
The cold and warm phases are different machines. Cold runs on a **cadence** (fixed waits, a
system-computed next-touch); warm runs on **next steps** (one explicit committed action per
deal) and on **opportunity stages**. A warm deal on a fixed nurture cadence is wrong, and a cold
lead with a hand-set "next step" is wrong. Keeping them as siblings keeps each model clean — and
mirrors the real handoff (`reply → Engaged`) in the funnel.

## How to edit
1. Edit `SKILL.md` (or ask Claude to).
2. To change *what it does*, edit the body. To change *when it fires*, edit the `description`
   in the frontmatter.
3. If the opportunity commands or Close setup change, update `SKILL.md`'s command surface to match
   (the canonical build spec lives with the CLI devs).
4. Keep it tight and self-contained; the inline command semantics are what let the agent avoid
   `--help`.

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update the
plugin from this marketplace (see the repo-root `README.md`).
