# lead-touch — skill source

This is the `lead-touch` skill, shipped inside the **`dogfu`** plugin. It covers the whole
post-research lifecycle of a lead in the Close CRM — the **cold** outreach cadence *and* the
**warm** deal phase — and triggers whenever a BDR wants to interact with the CRM about leads
that already exist: check a lead's status or cadence position, pull the leads due for a
follow-up, record a touch they just sent, mark a lead replied / not interested, run a
discovery call, open or advance or close an **opportunity**, set a deal's next step, intake
an inbound lead, or report the funnel / pipeline. It's the layer that runs **after**
`lead-research`: research adds and qualifies a lead; lead-touch works it through outreach and
the deal to a close.

The lifecycle it owns:

```
lead-research   →   lead-touch (cold flow)      →   lead-touch (warm flow)
(Discover &         (cadence: reach-out →           (Connected → discovery call →
 Qualify)            follow-ups; reply →             opportunity; Engaged →
                     Connected)                      Won/Customer or lost)
```

## Files
- `SKILL.md` — the shared operating manual: how our leads are organized in Close (status vs
  outreach state vs opportunity), the CLI-owned task discipline (`[dogfu:cadence]`,
  `[dogfu:engage]`, `[dogfu:deal:<opp_id>]` — single writer), the one `crm worklist` queue,
  how to run `dogfu`, the shared command surface (leads / contacts / notes / tasks), the
  shared operating rules — and the routing that tells the agent which flow reference to load.
  The YAML frontmatter `name` / `description` controls when the skill triggers.
- `references/cold-outreach.md` — the cold flow: the touch model (touch = attempt, channel =
  freeform label, no touch cap), the system-managed outreach fields and the "due" rule, touch
  vs task, the cold work queue, the `touch record / reply / stop` verbs, and the funnel /
  outreach-load report.
- `references/deals.md` — the warm flow: lead status vs opportunity, the qualification gate
  (open an opportunity only after a call confirms a real deal — never on a reply), the
  pipeline (Discovery → Trial → Proposal → Won/Lost), the engage/deal task model, the warm
  work queue, the `opportunity` verbs, inbound intake, and the pipeline / forecast report.

## Why one skill, two references
The cold and warm phases are different machines — cold runs on a **cadence** (fixed waits, a
system-computed next touch); warm runs on **next steps** (one explicit committed action per
deal) and **opportunity stages**. A warm deal on a nurture cadence is wrong, and a cold lead
with a hand-set "next step" is wrong. They used to be two sibling skills (`lead-touch` /
`lead-engage`), but the shared foundation (lead anatomy, task discipline, CLI setup, command
surface) was one model split across two files. Now `SKILL.md` holds that shared model and
routes to a per-flow reference, so each flow's machine stays clean and the agent loads only
the flow it's working — both, at the seam (a reply landing).

## The `dogfu` CLI dependency
Same dependency as `lead-research`: the skill is the orchestration layer and shells out to
the `dogfu` CLI for all data and CRM actions. The skill **does not bundle `dogfu`** — it's a
published package (`pip install dogfu`) installed and authenticated through the **dogfu MCP**
(`get_setup_instructions` → `dogfu configure --otp <OTP> --title "…"`). CRM (Close) calls go
through the backend under the Close API key set once in the admin **Console → CRM
Integration**. See the **Running `dogfu`** section of `SKILL.md`.

## The cadence lives in the CLI, not here
The touch schedule — the between-touch waits, and how the next-touch reminder is computed —
is defined in the CLI's own config and applied automatically by `dogfu crm touch record`.
Touch 0 is the reach-out, touches 1..N are follow-ups (unbounded), the channel is freeform
metadata recorded per touch, and the only exits are `reply`/`stop`. The references document
this so the agent can reason about it — but the CLI is the source of truth. If the config or
flags change, update the references to match.

## Status — CLI shipped; pending Close-side setup
The **`dogfu crm opportunity …`** commands have shipped and the **`Engaged`** lead status is
live (with `Trial` removed). Two Close-side admin steps remain: creating the **`Connected`**
lead status (the pre-gate reply state that drives the auto `[dogfu:engage]` task — until it
exists, `touch reply` ends the cadence but leaves the status unchanged, safely), and the
**opportunity pipeline** (Discovery → Trial → Proposal → Won/Lost) + **`Deal Type`** custom
field. Until those exist, the skill can read/move lead **statuses** but can't yet work
opportunities.

## How to edit
1. Edit `SKILL.md` or the flow reference (or ask Claude to).
2. To change *what it does*, edit the body — shared model in `SKILL.md`, flow mechanics in
   the matching `references/` file. To change *when it fires*, edit the `description` in the
   frontmatter.
3. Keep the shared/flow split honest: anything both flows need goes in `SKILL.md`; anything
   only one flow needs goes in its reference. The inline command semantics are what let the
   agent avoid `--help`.
4. If the opportunity commands or Close setup change, update `references/deals.md`'s command
   surface to match (the commands live in the `dogfu` CLI, in the berlin backend repo).

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update the
plugin from this marketplace (see the repo-root `README.md` for install steps).
