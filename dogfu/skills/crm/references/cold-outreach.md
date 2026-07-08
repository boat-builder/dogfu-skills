# Cold outreach — the cadence flow

This reference covers the **cold flow** of the crm skill: leads being chased through the
outreach sequence, from the first reach-out until they reply or the BDR stops. It assumes
the shared model and `dogfu` setup in `SKILL.md` and the shared command surface (lead
anatomy, reads, record editing) in `records.md`.

***

## The outreach model — a touch is an *attempt*, channel is a *label on it*

A lead is worked as a sequence of **touches**. A **touch is one attempt to reach the lead** —
recorded *after* the BDR sends it. The number is the *attempt order*:

- **Reach-out** — the **first** attempt to make contact (internally touch `0`). On **any**
  channel the BDR chooses, and it can be a **light touch** — a connection request, a like, or
  a comment — not only a DM or email.
- **Follow-up 1, 2, 3, … N** — each subsequent attempt (touch `1..N`). Also on any channel.
  **There is no fixed limit** — a deal that's too good to ignore can be nudged as many times
  as the BDR wants. The sequence does **not** auto-end.

**Channel is freeform metadata recorded on each touch.** The BDR records which channel they
actually used (if they tell you) — LinkedIn, email, X, a call, whatever. It's logged so they
can later see *how* we reached out at each step and *which channels are still untried*.
Channel is **optional**: record the touch with or without it.

Each touch can also carry an **optional free-text detail** — the message they sent, or any
context. Never mandatory; omit it if they don't give you one.

### The system-managed fields

You never set these by hand; the `touch` verbs compute them:

- **`touch_stage`** — count of **completed** touches, as a 0-based index of the last one.
  `null` = no touch yet (reach-out still pending). `0` after the reach-out, `1` after
  follow-up 1, and up with no ceiling.
- **`last_touched`** — date the most recent touch went out.
- **`next_touch_due`** — date the next follow-up is due. **Empty = the lead has left the
  sequence** (replied or stopped) and drops out of the follow-up queue.
- **`touch_channel`** — the **set of channels used so far** (multi-value: e.g.
  `["linkedin","email"]`) — *channels tried* for the lead. The channel of each *individual*
  touch lives in that touch's history entry.

**The "due" rule** (the heart of the work queue): a lead is due for a **follow-up** when
`next_touch_due ≤ today`. A lead is due for a **reach-out** when it's **Qualified and has no
touch yet** (`touch_stage` is null). The work queue is the union of those two — see below.

**Lifecycle:** `Qualified, untouched → reach-out → follow-up 1 → 2 → … (no cap)`. A **reply**
or a **stop** at any point ends the sequence (clears `next_touch_due`), which is what keeps a
lead who answered — or one we've decided to drop — from being messaged again. The sequence
never ends on its own — no touch cap, no timeout — so `stop` is a deliberate, first-class
action: it's how the BDR says "I'm done chasing this one."

**Touches never change status, and status moves don't advance the sequence** — though moving
a lead **out of the cold flow** (Connected / Engaged / Customer / a dead status) ends the
sequence (clears `next_touch_due`, closes the cadence task), and `touch record` refuses a
lead whose status has already left it. To end a chase, use `touch reply` / `touch stop` —
they record the *intent*, not just the label.

***

## Touch vs Task — keep them distinct

| | **Touch** | **Task** |
| :-- | :-- | :-- |
| Tense | **Past** — "I reached out / followed up" | **Future** — "do X by this date" |
| Is | A logged *event* (history / audit) | A *reminder* that drives the work queue |
| Carries | touch #, date, channel?, optional detail | due date, text, assignee |
| Created | *after* the BDR does the outreach | *before* — it's the to-do |

They're two faces of one step: **recording a touch closes the reminder that prompted it and
schedules the next one.** The touch is the receipt; the task is the ticket. The reminder here
is the **`[dogfu:cadence]` task** (see the task discipline in `SKILL.md`) — the reach-out is
simply the first cadence task, the follow-ups the ones after it, exactly one open per lead,
and only the CLI writes it. **The reach-out and every follow-up stay *touches* (events); the
forward reminder is the cadence *task*; discretionary one-offs are *ad-hoc tasks*. Nothing
else becomes a task.**

***

## The cold work queue — one list, two kinds of action

When the BDR is working the cadence, **run `dogfu crm worklist`** and read its cold rows,
most-overdue first, each labelled with its `next_action`:

1. **Reach-outs** (`--kind reach-out`) — Qualified leads with **no touch yet** (`touch_stage`
   null). "New people to say hello to."
2. **Follow-ups** (`--kind follow-up`) — leads whose next cadence touch is due, labelled with
   which follow-up is next and the channels already tried (so the BDR can vary channel).

`crm worklist` reads the actual open cadence tasks due today, so it and Close's native
pending-task list are literally the same list. (Ad-hoc tasks the BDR made outside the cadence
show up too, as `ad-hoc` rows — real work, never invisible.)

> **The task list is complete and trustworthy.** Every reach-out and follow-up is a real
> Close task, so a BDR can work straight from Close's own Tasks/Inbox, and **`crm
> reconcile`** is the backstop that keeps it honest (it flags a Qualified lead whose
> reach-out task never got created, a stray task after exit, and so on).

***

## The command surface — cadence reads & touch verbs

Shared reads and record editing are in `records.md`; these are the cold-flow additions.

### Reading cadence state

| Goal | Command |
| :-- | :-- |
| **The cold queue** — reach-outs + follow-ups due now | `dogfu crm worklist --kind reach-out` · `--kind follow-up` |
| **Outreach-load summary** (reach-out vs follow-up cadence tasks due + by-touch distribution) | `dogfu crm touch report` |
| A lead's **touch history** (each touch: #, date, channel, detail) | `dogfu crm touch history <lead_id> [-l N]` |

### Recording outreach & moving leads through the sequence

| What happened | Command | Effect |
| :-- | :-- | :-- |
| Sent the **next** touch (reach-out or follow-up) | `dogfu crm touch record <lead_id> [-c <channel>] [--detail "<text>"]` | auto-advances to the lead's next touch; stamps `last_touched`=today, computes `next_touch_due`, **adds `<channel>` to the channels-tried set** (if given), appends a structured touch-history entry. **Closes the prior cadence task and opens the next.** Channel and detail are **optional**. |
| Sent a touch with a **non-default wait** | add `--wait-days N` | overrides the gap before the next follow-up is due |
| Recorded a touch but **don't** want the auto note/task | add `--no-note` and/or `--no-task` | suppresses the side effect, at a cost (each warns): `--no-note` → the touch won't show in `touch history`; `--no-task` → the lead is due but invisible to `crm worklist` until reconcile repairs it. To end the sequence use `--final`. |
| Lead **replied / positive** | `dogfu crm touch reply <lead_id>` | ends the sequence (clears `next_touch_due`), moves the lead to **Connected** automatically, and opens its **`[dogfu:engage]` task** ("land the discovery call") — the **handoff out of cold** (see below). Pass `--status <id>` only to override. **Never downgrades**: a lead already Connected / Engaged / Customer keeps its status (warned, cadence still ends). |
| **Giving up** | `dogfu crm touch stop <lead_id> [--status <Bad Fit or Not Interested id>]` | same mechanism as reply; intent + status differ. Without `--status` the output **warns if the lead is left in a workable status** (limbo) — set the real terminal status. |
| Change **status only** | `dogfu crm lead update <lead_id> -s <status_id>` | funnel label — plus the safety net: moving out of the cold flow (Connected / Engaged / Customer / dead) also **ends the sequence** (clears the due date, closes the cadence task) |

Notes on the verbs:

- `record` **auto-advances** from the lead's current touch — on a never-touched lead it logs
  the **reach-out**; otherwise the **next follow-up**. Don't force a number unless you mean
  to redo a touch, and don't record the same touch twice (check `touch_stage` if unsure).
- `record` **errors on a lead that has left the cold flow** (Connected / Engaged / Customer /
  a dead status) — recording there would re-enroll it. If the BDR really means to re-chase,
  first move it back to a working status (`lead update -s <Qualified id>`), then record.
- **Channel is freeform and optional.** Pass `-c linkedin|email|x|call|other` (or whatever
  the BDR used) when they tell you; omit it when they don't. It's added to the lead's
  channels-tried set and stamped on that touch's history entry.
- **Detail is optional.** Pass `--detail "<message or context>"` to store what they actually
  sent; omit when there's nothing to add.
- `record` defaults to **logging a note and creating the next-touch reminder task** — that's
  the BDR's automatic reminder and what `touch history` reads back. Leave them on unless
  asked.
- `record` does **not** change status (status stays a human label). `reply` moves the lead to
  **Connected** on its own; pass `--status` on `reply`/`stop` (or `lead update -s`) only to
  set a *different* status.
- `reply` vs `stop`: both end the sequence and pull the lead from the follow-up queue.
  `reply` = they answered (→ **Connected**, set automatically, with its engage task opened);
  `stop` = we're done (→ Bad Fit or Not Interested via `--status`). The difference is intent.
  Since nothing auto-ends the sequence, **`stop` is how a chase ends.**
- **A `reply` → Connected is the handoff out of this flow.** The cold cadence is done with
  the lead; the **deals flow** (`deals.md`) works it from there — first landing
  the discovery call (the auto engage task), then, once a deal is confirmed, opening an
  opportunity (→ **Engaged**) and running it (discovery → trial → proposal → won/lost). Don't
  keep recording touches on a Connected lead — it has left the cold cadence. (A trial is a
  deal *stage* tracked as an opportunity, not a cold-outreach outcome — there's no Trial lead
  status.)

***

## Reporting the funnel / outreach load

Two parts: the **outreach load** has a native command; the **funnel by status** you compose.

1. **Outreach load due today — `crm touch report`.** It returns reach-outs-due vs
   follow-ups-due and the follow-up distribution by next touch number (Follow-up 1 / 2 / 3+),
   computed for you — don't re-derive it by hand. It's task-grounded, so the counts agree
   with `crm worklist` (the actionable list behind them); a reach-out *owed but not yet
   tasked* is a `crm reconcile` gap.
2. **Funnel by status** (no aggregate endpoint — compose it): `crm status list` → the ids,
   then per status `crm lead list -s <id> -l <high> -o file` and count → leads in each funnel
   status.

Lead with the **number that needs work today** (from `touch report`), then a funnel table
(status → count) and the outreach table (reach-outs due, follow-ups due). **Per-status counts
are capped by `--limit`** — for a big funnel, raise the limit or report "at least N" and say
so.

***

## Cold-flow operating rules

(On top of the shared rules in `SKILL.md`.)

- **Channel and detail are optional but valuable.** Log the channel whenever the BDR mentions
  it, and the message/detail when they share it — that's what lets them later see how each
  touch went and which channels are still worth trying. Never block on them.
- **Never assume a reply.** Close can't see LinkedIn/X replies. Only run `touch reply` when
  the BDR states the lead responded.
- **`stop` ends a chase.** Nothing auto-ends the sequence, so when the BDR is done with a
  lead, `touch stop` it (with the right `--status`) — don't just let it sit due forever.
- **Don't touch the cadence task by hand.** Let `record`/`reply`/`stop` manage the one system
  reminder.
- **Don't double-record.** `record` advances from the current touch; running it twice skips
  one. Check `touch_stage` if unsure.

## Response format

- **Work queue:** a numbered / tabular list — lead name + company, the contact(s) with their
  LinkedIn/X links, last touched, days overdue, **channels already tried**, and the **next
  action** (Reach-out, or Follow-up N — suggest a fresh channel if useful). Make it
  copy-paste actionable so the BDR can go message immediately.
- **After a write:** one line confirming what changed — lead, the touch just logged (with
  channel if given), and the next due date (or "sequence ended").
- **Status/report:** the headline "needs work today" number first, then the funnel + outreach
  tables.
