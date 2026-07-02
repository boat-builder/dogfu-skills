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
  cadence and hands a lead off the moment it replies (→ **Connected**); lead-engage takes it from
  there and works it to a close. For the cross-phase worklist of everything to work today ("what
  should I work on", any or all stages, read-only, with context) use `lead-worklist`; for cold
  prospecting cadence use `lead-touch`; for researching or qualifying a brand-new target use
  `lead-research`.
---

# Lead engage — work live deals in Close (powered by `dogfu`)

This skill runs the **warm phase** — the work after a lead answers. `lead-touch` hands a lead
off the moment it replies (→ **Connected**); `lead-engage` takes it from there to a close:
the discovery call, an opportunity, a trial, a proposal, won or lost.

The model is two ideas: **a lead's status is its lifecycle label; an opportunity is the actual
deal you forecast.** Everything below follows from that split.

***

## Scope & handoffs

You work leads in the two warm statuses — **Connected** (they replied / you're in a live
conversation, *no deal confirmed yet*) and **Engaged** (a real deal is open) — and their **open
opportunities**. A lead lands here two ways: a cold lead **replied** (`lead-touch` set it
**Connected**), or it came in **inbound** (you create it Connected — see *Inbound intake*). It
leaves when the deal is **Won** (→ Customer) or **lost**.

Stay in lane: cold-outreach cadence is `lead-touch`, researching a brand-new target is
`lead-research`, and a read-only health audit (dropped balls, drifted statuses) is `crm-cleanup`.

***

## The two layers — Lead status vs Opportunity (the core model)

Hold these apart; everything else follows from the split.

- **Lead status** = the **account's lifecycle label** (one per lead). The funnel:
  *Potential → Qualified → **Connected → Engaged** → Customer*, with *Bad Fit / Not Interested /
  Canceled / DNC* as terminal labels. Account-specific ids — resolve them live with `crm status
  list`, never hardcode. The two warm labels mean different things: **Connected** = replied, in
  conversation, *no deal yet*; **Engaged** = a real deal (opportunity) is open. A lead sits at
  Engaged for the entire active deal and only flips to **Customer** when it's won.
- **Opportunity** = the **deal itself** — the thing you forecast. It lives *on* a lead and
  carries a **pipeline stage**, a **value** (recurring MRR), a **deal type** (Co-Pilot vs
  Fully-Run), and a **confidence**. A lead usually has one open opportunity, but can carry more
  than one (e.g. a Co-Pilot *and* a Fully-Run deal) — each its own opportunity with its own stage
  and next step.

So a warm lead is in one of two states, and its **status mirrors which**:
- **Connected, no opportunity** — it replied / came inbound, but no call has confirmed a real
  deal. The job is to *land the discovery call* (driven by the auto `[dogfu:engage]` task).
- **Engaged, one or more open opportunities** — a call confirmed a deal; it's tracked and moving
  through the pipeline. The job is to *advance the stage*.

**Customer** = the lead has a **Won** opportunity. Don't pre-flip a lead to Customer because a
trial started — that's a *Trial-stage opportunity*, still Engaged.

***

## The qualification gate — when to open an opportunity (NOT on a reply)

**A reply is a conversation, not a deal.** A reply makes a lead **Connected**, not a deal — it
can be "who are you" or "maybe later", and a deal per reply makes the forecast meaningless.
**Open the opportunity only when a call has confirmed a real deal** — a defined *need*, a
plausible *buyer*, and a realistic *path*. That call is the gate.

- **Before the gate** (**Connected**, no opp): the work is to land the discovery / intro call.
  It's already driven by the auto-created **`[dogfu:engage]` task** ("Land the discovery call") —
  the CLI opens it the moment the lead becomes Connected, so you don't create it by hand.
- **At the gate** (call confirms a deal): **open the opportunity** (`crm opportunity create`) at
  the *Discovery* stage, with the value and deal type you scoped. This **promotes the lead
  Connected → Engaged and retires the engage task automatically** (the deal task takes over).
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

## The unit of work: one CLI-owned next-action task per stage

Every warm stage carries **exactly one open, CLI-owned next-action task** — the same
single-writer discipline as the cold cadence task (`[dogfu:cadence]`). Two tags cover this phase:

- **`[dogfu:engage]` — the pre-gate task (Connected).** The moment a lead becomes Connected
  (a reply, or inbound intake), the CLI opens **exactly one** engage task, "Land the discovery
  call", due today. It's **CLI-owned** — opened on entry, retired automatically when the lead
  leaves Connected (a deal opens → Engaged, or it goes terminal). **Never hand-create or
  hand-complete it.** You clear it by clearing the state: open the opportunity (the gate) or set
  a terminal status. (If a prospect says "call me next week", you *can* reschedule the reminder
  with `crm task update <task_id> -d <date>` — reconcile enforces that the task *exists*, not its
  due date.)
- **`[dogfu:deal:<opp_id>]` — the deal task (Engaged).** Each open opportunity carries **exactly
  one open next-step task** (the opp id is in the tag, so a lead's separate deals keep separate
  next steps; `[dogfu:deal]` is the shared prefix). Unlike the engage task, its text and due date
  are a *human decision* ("send the proposal Friday", "wait for their board Tue") — you author
  them, and the **opportunity verbs are the only writer**: setting a new next step closes the
  prior one, and `win`/`lose` close it when the deal exits. **Never hand-create or
  hand-complete a deal task**; use `crm opportunity next`.
- **A live deal with no open next step is a dropped ball**, and so is a Connected lead with no
  engage task — surface either loudly and fix it. (`crm reconcile` audits both invariants across
  the whole book and can auto-repair them.)

***

## The engage work queue — "what do I move today?"

The daily question is "what needs me today?" Because both warm stages now carry a real task, the
whole phase shows up in the one work queue:

1. **Discovery calls to land** — Connected leads' engage tasks due ≤ today:
   **`dogfu crm worklist --kind engage`** (each row's `next_action` is "Land the discovery call").
   *Book the call, then open the opportunity — the gate.*
2. **Deal next-steps due** — open opps whose next-step task is due ≤ today:
   **`dogfu crm worklist --kind deal`** (each row carries `next_action` + `opportunity_stage`).
   *Act now.*
3. **Dropped balls** — a Connected lead with **no engage task**, or an open opp with **no
   next-step task**. There's no task, so they aren't in the inbox — **`dogfu crm reconcile`**
   surfaces them (`missing_engage_task` / `opportunity_no_next_step`) and can auto-repair them.
4. **Stalled deals** — open opps not updated in > 14 days, also from **`crm reconcile`**
   (`stalled_opportunity`). *Quietly dying; nudge or re-qualify.*

So: `crm worklist` (kinds `engage` + `deal`) is the "act today" list across the whole warm phase,
and `crm reconcile` catches whatever fell out of it (a missing task, or a deal gone stale) — the
pre-gate conversation that stalls no longer slips, because it's a first-class task, not an absence
you have to hunt for.

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

- **First, authenticate the CLI.** Before any `dogfu` command, call the dogfu MCP's
  **`get_setup_instructions`** tool and follow it: install the CLI (if needed), then
  `dogfu configure --otp <OTP> --title "Lead engage: <who/what>"` with the OTP it returns. This
  is the first step whenever the skill runs.
- **Output is JSON by default** — read stdout. `-o FILE` dumps a large list to a file; `-f
  table` is for humans, not you.
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
| Lead statuses + ids (do first; resolve Connected / Engaged / Customer / etc.) | `dogfu crm status list` |
| Opportunity pipelines & their stage statuses + ids (do first for deals) | `dogfu crm opportunity pipelines` |
| Connected leads (pre-gate: replied, no deal yet) | `dogfu crm lead list -s <Connected id>` |
| Engaged leads (deals open) | `dogfu crm lead list -s <Engaged id>` |
| Full lead incl. contacts | `dogfu crm lead get <lead_id>` |
| A lead's open/closed opportunities | `dogfu crm opportunity list [-l <lead_id>] [--status open]` |
| One opportunity in full | `dogfu crm opportunity get <opp_id>` |
| **Discovery calls to land today** (engage tasks due, in the one work queue) | `dogfu crm worklist --kind engage [-l N]` |
| **Deals due today** (next-steps due, in the one work queue) | `dogfu crm worklist --kind deal [-l N]` |
| **Dropped-ball & stalled work** (missing engage task, or open opp with no next-step / gone stale) | `dogfu crm reconcile` (rows `missing_engage_task` / `opportunity_no_next_step` / `stalled_opportunity`) |
| A lead's notes / tasks | `dogfu crm note list <lead_id>` · `dogfu crm task list -l <lead_id> [-p]` |

### The gate & moving a deal

| What happened | Command | Effect |
| :-- | :-- | :-- |
| **Discovery call confirmed a real deal** (the gate) | `dogfu crm opportunity create <lead_id> --value <mrr> --period monthly --deal-type co-pilot\|fully-run [--confidence N] [--next "<action>" --next-due YYYY-MM-DD] --note "<scope>"` | opens the opportunity at **Discovery** and (with `--next`) its first `[dogfu:deal]` task; **promotes the lead Connected → Engaged and retires the engage task automatically**. |
| They **started a trial / POC** | `dogfu crm opportunity update <opp_id> --stage trial` | moves the deal to **Trial** |
| **Sent the proposal / terms** | `dogfu crm opportunity update <opp_id> --stage proposal` | moves the deal to **Proposal** |
| **Signed / closed-won** | `dogfu crm opportunity win <opp_id> [--value <final>] [--close YYYY-MM-DD]` then `dogfu crm lead update <lead_id> -s <Customer id>` | marks the opp **Won**; set lead status **Customer** |
| **Dead / lost** | `dogfu crm opportunity lose <opp_id> [--reason "<why>"]` then `dogfu crm lead update <lead_id> -s <Not Interested\|Bad Fit id>` | marks the opp **Lost**; set the terminal lead status |
| **Customer churned** | `dogfu crm opportunity lose <opp_id> --reason "churn: <why>"` then `lead update <lead_id> -s <Canceled id>` | for an existing Customer |
| **Log what happened** (call, demo, trial start, proposal) | `dogfu crm note create <lead_id> -t "<event + outcome>"` | audit trail |
| **Set / replace the next step** (open opp) | `dogfu crm opportunity next <opp_id> -t "<action>" -d <YYYY-MM-DD>` | swaps the single open `[dogfu:deal]` task (CLI is the single writer — don't touch it by hand) |
| **Pre-gate next step** (Connected, no opp) | *nothing to do — the `[dogfu:engage]` task is auto-opened* | To snooze the reminder (e.g. "call me next week"): `dogfu crm task update <task_id> -d <YYYY-MM-DD>`. Don't hand-create/complete it; it clears when you open the opp or set a terminal status. |
| Update deal economics as it firms up | `dogfu crm opportunity update <opp_id> [--value] [--period] [--confidence] [--deal-type] [--note]` | adjust value / confidence / deal type |

### Inbound intake

An inbound lead (someone reached out to us) skips the cold sequence entirely:

1. `dogfu crm lead create -n "<company>" -u <url> -d "INBOUND (<channel>, <date>): <one-line>" -s <Connected id>` — created straight into **Connected**, which auto-opens its engage task ("land the discovery call").
2. `dogfu crm contact create <lead_id> -n "<name>" -t "<title>" -u <linkedin> [-e <email>]` for each person.
3. `dogfu crm note create <lead_id> -t "<intro-call notes / what they want / deal shape>"`.
4. If that first call already **confirmed a deal**, open the opportunity (the gate command above) — that promotes the lead to Engaged and retires the engage task in one motion. Otherwise the engage task is already there to drive landing the qualifying call.

> If the inbound company hasn't been researched, consider a `lead-research` pass first to fill
> the curated fields and ICP read — but **don't block the conversation on it**; intake as
> Connected now, enrich after.

***

## Operating rules

- **Never open an opportunity on a reply.** A reply makes a lead **Connected**; wait for the call
  that confirms a real deal to open the opp (which promotes it to Engaged). Connected ≠ a deal.
- **One open opportunity per active deal; one open `[dogfu:deal]` task per open opportunity; one
  open `[dogfu:engage]` task per Connected lead.** If you find a Connected lead with no engage
  task, or an open deal with no next step, that's the first thing to fix — but let the CLI be the
  single writer of both tags (`opportunity` verbs for `[dogfu:deal]`; the reply/gate/status
  transitions for `[dogfu:engage]`). **Never hand-create or hand-complete either.**
- **The lead stays Engaged for the whole live deal.** Flip to **Customer** only on win; to
  **Not Interested / Bad Fit** only on loss; **Canceled** only on churn. Don't leave a zombie
  Engaged lead after a deal dies — losing the opp **and** setting the terminal status go
  together. (A Connected lead that dies before the gate goes straight to a terminal status, which
  retires its engage task.)
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

1. `crm opportunity list --status open` → every open deal with stage + value.
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
