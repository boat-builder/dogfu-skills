---
name: lead-engage
description: >-
  Work Agent Berlin's *live deals* in the Close CRM — the warm phase that begins once a lead
  engages — using the `dogfu` CLI. Use this whenever someone wants to act on a lead that has
  already replied or come in inbound: run a discovery / intro call, decide whether there's a
  real deal, open or advance or close an **opportunity** (Discovery → Trial → Proposal →
  Won/Lost), log a call / demo / proposal that just happened, set or check the **next step** on
  a deal, pull the deals that need action today or have stalled, move a deal to Trial or
  Proposal, mark a deal won or lost, intake an inbound lead, or report the deal pipeline /
  forecast. This is the counterpart to `lead-touch`: lead-touch runs the *cold* outreach
  cadence and hands a lead off the moment it replies (→ **Engaged**); lead-engage takes it from
  there and works it to a close. For cold prospecting cadence use `lead-touch`; for researching
  or qualifying a brand-new target use `lead-research`.
---

# Lead engage — work live deals in Close (powered by `dogfu`)

This skill runs the **warm phase** — the work after a lead engages. `lead-touch` hands a lead
off the moment it replies (→ **Engaged**); `lead-engage` takes it from there to a close:
discovery, an opportunity, a trial, a proposal, won or lost.

The model is two ideas: **a lead's status is its lifecycle label; an opportunity is the actual
deal you forecast.** Everything below follows from that split.

***

## Scope & handoffs

You work leads in **Engaged** status and their **open opportunities** — the live-deal slice of
the funnel. A lead lands here two ways: a cold lead **replied** (`lead-touch` set it Engaged),
or it came in **inbound** (you create it Engaged — see *Inbound intake*). It leaves when the
deal is **Won** (→ Customer) or **lost**.

Stay in lane: cold-outreach cadence is `lead-touch`, researching a brand-new target is
`lead-research`, and a read-only health audit (dropped balls, drifted statuses) is `crm-cleanup`.

***

## The two layers — Lead status vs Opportunity (the core model)

Hold these apart; everything else follows from the split.

- **Lead status** = the **account's lifecycle label** (one per lead). The funnel:
  *Potential → Qualified → Engaged → Customer*, with *Bad Fit / Not Interested / Canceled /
  DNC* as terminal labels. Account-specific ids — resolve them live with `crm status list`,
  never hardcode. In this phase a lead sits at **Engaged** for the entire active deal and
  only flips to **Customer** when the deal is won.
- **Opportunity** = the **deal itself** — the thing you forecast. It lives *on* a lead and
  carries a **pipeline stage**, a **value** (recurring MRR), a **deal type** (Co-Pilot vs
  Fully-Run), and a **confidence**. A lead usually has one open opportunity, but can carry more
  than one (e.g. a Co-Pilot *and* a Fully-Run deal) — each its own opportunity with its own stage
  and next step.

So an Engaged lead is in one of two states:
- **No opportunity yet** — it replied / came inbound, but no call has confirmed a real deal.
  The job is to *get the qualifying call to happen*.
- **One or more open opportunities** — a call confirmed a deal; it's tracked and moving through
  the pipeline. The job is to *advance the stage*.

**Customer** = the lead has a **Won** opportunity. Don't pre-flip a lead to Customer because a
trial started — that's a *Trial-stage opportunity*, still Engaged.

***

## The qualification gate — when to open an opportunity (NOT on a reply)

**A reply is a conversation, not a deal.** Don't open an opportunity just because a lead is
Engaged — a reply can be "who are you" or "maybe later", and a deal per reply makes the forecast
meaningless. **Open the opportunity only when a call has confirmed a real deal** — a defined
*need*, a plausible *buyer*, and a realistic *path*. That call is the gate.

- **Before the gate** (Engaged, no opp): the work is to land the discovery / intro call.
  Drive it with a **next-step task** ("book intro call", "follow up to schedule").
- **At the gate** (call confirms a deal): **open the opportunity** at the *Discovery* stage,
  with the value and deal type you scoped on the call.
- **Inbound shortcut:** an inbound lead that booked a call about a *specific* deal often
  clears the gate on that first call — discovery and opening the opportunity happen in one
  motion. (That's still "qualified deal → opp", not "reply → opp".)

***

## The pipeline: Discovery → Trial → Proposal → Won / Lost

What each stage means and a typical next step:

| Stage | The deal is… | Typical next step |
| :-- | :-- | :-- |
| **Discovery** | opportunity just opened; scoping the need and the engagement shape (Co-Pilot vs Fully-Run), agreeing a trial/POC | "set up the trial", "send scope" |
| **Trial** | they're running a trial / POC of Berlin | "check in mid-trial", "review results on <date>" |
| **Proposal** | trial validated; terms and pricing are out, awaiting a decision | "follow up on proposal", "negotiate terms" |
| **Won** *(terminal)* | signed → set lead status **Customer** | onboarding (out of scope here) |
| **Lost** *(terminal)* | dead → set lead status **Not Interested** / **Bad Fit** (or **Canceled** if a customer churned) | — |

**Value, deal type, and confidence ride on the opportunity, not the stage** — set them when you
open the deal and update them as it firms up.

***

## The unit of work: the **next step** (not a cadence)

Cold leads run on a **cadence** — fixed waits between touches the CLI computes (that's
`lead-touch`, whose single rolling reminder is tagged `[dogfu:cadence]`). Live deals can't run
on a wait curve: a deal's next step is a *human decision* ("send the proposal Friday", "wait
for their board meeting Tue"), not a computed gap. So you set it explicitly — but it follows the
**same single-writer pattern** as the cadence task.

- **The next-step task is CLI-owned and single-writer**, the deal-phase analogue of the cadence
  task: each open opportunity carries **exactly one open next-step task, tagged
  `[dogfu:deal:<opp_id>]`** (the opportunity id is in the tag, so a lead's separate deals keep
  separate next steps; `[dogfu:deal]` is the shared prefix). You author its text and due date;
  the **opportunity verbs are the only writer** — setting a new next step closes the prior one,
  and `win`/`lose` close it when the deal exits. **Never hand-create or hand-complete a deal
  task** (same rule as `[dogfu:cadence]`); use `crm opportunity next`.
- **Before the gate** (Engaged, no opportunity yet) there's no opportunity to hang it on, so the
  one next step — *land the qualifying call* — is an ordinary **ad-hoc task** (`crm task
  create`). Opening the opportunity is itself that step's completion.
- **A live deal with no open next step is a dropped ball** — surface it loudly and set a next
  step. (`crm-cleanup` audits this invariant across the whole book.)

***

## The engage work queue — "what do I move today?"

The daily question is "which deals need me today?" `dogfu crm opportunity due` returns the deal
queue in one list (the deal-phase analogue of `crm touch due`), most-urgent first, each row
tagged by why it's there:

1. **Due next-steps** — open opps whose next-step is due ≤ today. *Act now.*
2. **Dropped balls** — open opps with **no open next-step** (`next_step` is null). *Surface these
   loudest — set a next step.*
3. **Stalled deals** — open opps with no update in > 14 days. *Quietly dying; nudge or re-qualify.*

Read each row's `next_step` (null = dropped ball) and `date_updated` (staleness) to see why it's
in the queue. One case `opportunity due` can't catch — it's opportunity-scoped — is a **pre-gate
Engaged lead with no opportunity and no open task**; pull those from the Engaged lead list (`lead
list -s <Engaged id>`) so a conversation that stalls *before* the gate doesn't slip.

***

## Events → notes; stage moves → opportunity verbs

Two distinct kinds of write, don't conflate:

- **What happened** (a call, a demo, a trial kickoff, a proposal sent) → log a **note**
  (`crm note create`). It's the audit trail. **Don't** record these as "touches" — a touch is a
  cold-sequence concept owned by `lead-touch`.
- **The deal moved** (gate cleared, trial started, terms sent, signed, dead) → use the
  **opportunity verbs** to change the stage, then **reset the next step**.

A single interaction is usually: *note (what happened) → advance the opp stage if it moved →
set the new next step (`opportunity next`, which swaps the single `[dogfu:deal]` task).*

***

## Running `dogfu`

> **Where a flag's exact spelling depends on the CLI, follow `dogfu crm <cmd> --help`.**

Same dependency and auth as `lead-touch`. `dogfu` is a published CLI (`pip install dogfu`) on
your PATH:

```bash
dogfu crm <command> [flags]
```

- **Output is JSON by default** — read stdout. `-o FILE` dumps a large list to a file; `-f
  table` is for humans, not you.
- **Auth** is a `dogfu` session token via the **dogfu MCP**. On an auth error (`missing dogfu
  token`, `invalid or expired token`), call the MCP's **`get_setup_instructions`** and follow
  it (`dogfu configure --otp <OTP> --title "Lead engage: <who/what>"`), then retry.
- **CRM (Close) calls** use the Close API key set in the admin **Console → CRM Integration**;
  a missing key returns a `412` — surface it, don't retry.
- **Writes change external CRM state.** Before any create/update/delete, show the exact command
  (or a summary of what changes) and wait for an explicit go-ahead. Reads need no confirmation.
- Read calls are independent HTTP requests; run them concurrently when it helps.

***

## The command surface

Existing `dogfu crm` commands (statuses, leads, contacts, notes, tasks) work exactly as in
`lead-touch` — see that skill for their full flags. This skill adds the **opportunity** verbs.
All leaf commands take `-f json|table` (default json) and `-o FILE`.

### Reading state

| Goal | Command |
| :-- | :-- |
| Lead statuses + ids (do first; resolve Engaged / Customer / etc.) | `dogfu crm status list` |
| Opportunity pipelines & their stage statuses + ids (do first for deals) | `dogfu crm opportunity pipelines` |
| Engaged leads (the warm book) | `dogfu crm lead list -s <Engaged id>` |
| Full lead incl. contacts | `dogfu crm lead get <lead_id>` |
| A lead's open/closed opportunities | `dogfu crm opportunity list [-l <lead_id>] [--open]` |
| One opportunity in full | `dogfu crm opportunity get <opp_id>` |
| **The deal queue** (next-steps due now; stalled if supported) | `dogfu crm opportunity due [-l N]` |
| A lead's notes / tasks | `dogfu crm note list <lead_id>` · `dogfu crm task list -l <lead_id> [-p]` |

### The gate & moving a deal

| What happened | Command | Effect |
| :-- | :-- | :-- |
| **Discovery call confirmed a real deal** | `dogfu crm opportunity create <lead_id> --value <mrr> --period monthly --deal-type co-pilot\|fully-run [--confidence N] [--next "<action>" --next-due YYYY-MM-DD] --note "<scope>"` | opens the opportunity at **Discovery** and (with `--next`) its first `[dogfu:deal]` task. Lead stays **Engaged**. |
| They **started a trial / POC** | `dogfu crm opportunity advance <opp_id> --stage trial` | moves the deal to **Trial** |
| **Sent the proposal / terms** | `dogfu crm opportunity advance <opp_id> --stage proposal` | moves the deal to **Proposal** |
| **Signed / closed-won** | `dogfu crm opportunity win <opp_id> [--value <final>] [--close YYYY-MM-DD]` then `dogfu crm lead update <lead_id> -s <Customer id>` | marks the opp **Won**; set lead status **Customer** |
| **Dead / lost** | `dogfu crm opportunity lose <opp_id> [--reason "<why>"]` then `dogfu crm lead update <lead_id> -s <Not Interested\|Bad Fit id>` | marks the opp **Lost**; set the terminal lead status |
| **Customer churned** | `dogfu crm opportunity lose <opp_id> --reason "churn: <why>"` then `lead update <lead_id> -s <Canceled id>` | for an existing Customer |
| **Log what happened** (call, demo, trial start, proposal) | `dogfu crm note create <lead_id> -t "<event + outcome>"` | audit trail |
| **Set / replace the next step** (open opp) | `dogfu crm opportunity next <opp_id> -t "<action>" -d <YYYY-MM-DD>` | swaps the single open `[dogfu:deal]` task (CLI is the single writer — don't touch it by hand) |
| **Set the next step pre-gate** (Engaged, no opp) | `dogfu crm task create -l <lead_id> -t "<action, e.g. book intro call>" -d <YYYY-MM-DD>` | an ordinary ad-hoc task until the opportunity exists |
| Update deal economics as it firms up | `dogfu crm opportunity update <opp_id> [--value] [--period] [--confidence] [--deal-type] [--note]` | adjust value / confidence / deal type |

### Inbound intake

An inbound lead (someone reached out to us) skips the cold sequence entirely:

1. `dogfu crm lead create -n "<company>" -u <url> -d "INBOUND (<channel>, <date>): <one-line>" -s <Engaged id>` — created straight into **Engaged**.
2. `dogfu crm contact create <lead_id> -n "<name>" -t "<title>" -u <linkedin> [-e <email>]` for each person.
3. `dogfu crm note create <lead_id> -t "<intro-call notes / what they want / deal shape>"`.
4. If that first call already **confirmed a deal**, open the opportunity (the gate command above). Otherwise set an ad-hoc task (`task create`) to land the qualifying call.

> If the inbound company hasn't been researched, consider a `lead-research` pass first to fill
> the curated fields and ICP read — but **don't block the conversation on it**; intake as
> Engaged now, enrich after.

***

## Operating rules

- **Never open an opportunity on a reply.** Wait for the call that confirms a real deal. Engaged
  ≠ a deal.
- **One open opportunity per active deal; one open `[dogfu:deal]` task per open opportunity.** If
  you find an open deal with no next step, that's the first thing to fix. Let the `opportunity`
  verbs be the single writer of the `[dogfu:deal]` task — never hand-create or hand-complete it
  (same rule as `[dogfu:cadence]` in `lead-touch`).
- **The lead stays Engaged for the whole live deal.** Flip to **Customer** only on win; to
  **Not Interested / Bad Fit** only on loss; **Canceled** only on churn. Don't leave a zombie
  Engaged lead after a deal dies — losing the opp **and** setting the terminal status go
  together.
- **Put data where it belongs:** deal value / deal type / confidence → on the
  **opportunity**; what happened → a **note**; the forward reminder → the **`[dogfu:deal]` task**
  (via `opportunity next`); the person's LinkedIn/X → the contact's `urls`. Don't dump deal
  economics into the lead description.
- **Resolve ids at runtime** — `crm status list` for statuses, `crm opportunity pipelines` for
  stages. Never hardcode.
- **Never assume an event.** Close can't see LinkedIn/email/calls — only record a call, trial,
  reply, or signature when the human tells you it happened.
- **Surface stalled deals** (no stage move, no next step) every time you report the queue.

***

## Reporting the pipeline / forecast

No aggregate endpoint — compose from list calls:

1. `crm opportunity list --open` → every open deal with stage + value.
2. **Pipeline by stage:** count and sum value across *Discovery / Trial / Proposal*.
3. **Weighted forecast:** Σ (value × confidence) over open opps — the realistic number.
4. **Needs-action:** the deal queue (next-steps due) + **dropped balls** (no next step) +
   **stalled** (no stage move in > 14 days).
5. **Closed:** Won and Lost this period (with values) for win-rate.

Lead with the **deals that need action today**, then the stage table (stage → count → value),
then the weighted forecast. Note that counts are capped by `--limit` on big books.

***

## Response format

- **Deal queue:** a numbered / tabular list — company, current **stage**, **value**, the
  **next step + due date** (or a loud **"⚠ NO NEXT STEP"**), days in stage, and the contact(s)
  with their links. Make it copy-paste actionable.
- **After a write:** one line confirming what changed — the deal, the **stage it moved to** (or
  opened / won / lost), and the **next step you set**.
- **Pipeline / forecast:** the "needs action today" set first, then the stage table and the
  weighted forecast number.
