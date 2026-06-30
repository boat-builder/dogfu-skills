---
name: lead-touch
description: >-
  Manage Agent Berlin's leads in the Close CRM after the BDR has done the actual
  outreach — using the `dogfu` CLI. Use this whenever someone wants to interact with
  the CRM about leads that already exist: check a lead's status or where it stands in
  the outreach sequence, pull the leads to act on now ("who do I message today", "my
  work queue"), record a touch they just sent (a reach-out or a follow-up, on any
  channel), mark a lead as replied / engaged / not interested / nurture, change a
  lead's status, read a lead's touch history, add a note or contact or follow-up task
  to a lead, or report on the funnel / outreach load. The BDR sends the messages
  themselves; this skill is how they tell the CRM what happened and ask it what to do
  next. For researching or qualifying a brand-new target, use the lead-research skill
  instead — this is the layer that runs after that.
---

# Lead touch — CRM & outreach ops (powered by `dogfu`)

This skill is how a BDR **talks to the CRM about leads that are already in it**. The BDR
does the real-world outreach by hand — sends the LinkedIn request, the DM, the email, the X
message, whatever fits. Your job is to translate what they tell you into the right `dogfu
crm` calls: read where a lead stands, pull what's due, record what they sent, move a lead's
status, and keep notes/tasks/contacts straight. The `dogfu` CLI holds the Close credentials
and commands; you hold the model of how our leads are organized, below — so you rarely need
`--help`.

The work here is open-ended — far more than a fixed list of plays. Anything a BDR would do
in Close, they can ask you to do. The sections that follow are the complete mental model and
command set; reason from them rather than pattern-matching to a script.

***

## How our leads are organized in Close

A **lead is a company** (not a person). Each lead carries:

- **Identity** — `name`, `url` (website), `description` (a short headline).
- **Status** (`status_id` + `status_label`) — where it sits in the **funnel**. A human
  judgment label, **account-specific** — resolve the live ids with `crm status list`, never
  hardcode them. Typical labels: *Potential, Qualified, Engaged, Customer, Bad Fit, Not
  Interested, Canceled, DNC*.
- **Contacts** — the people you actually reach out to. Each has `urls` (their LinkedIn / X —
  what you DM from), `emails`, `phones`, `title`.
- **Outreach state** — system-managed fields tracking the outreach sequence (below).
- **Curated company fields** — `industry`, `employees`, `revenue`, `business_model`,
  `seo_pages`. Set by lead-research; you rarely touch them here.
- Attached **notes** (write-ups, audit trail) and **tasks** (follow-up reminders).

**Two layers — don't conflate them.** *Status* is the funnel position (a human label).
*Outreach state* is the sequence position (machine-tracked). They move independently:
recording a touch advances the sequence but **does not** change status; changing status
doesn't touch the sequence. Only `touch reply` / `touch stop` deliberately do both (end the
sequence **and** optionally set a status).

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

**Channel is freeform metadata recorded on each touch.** The BDR
records which channel they actually used (if they tell you) — LinkedIn, email, X, a call,
whatever. It's logged so they can later see *how* we reached out at each step and *which
channels are still untried*. Channel is **optional**: record the touch with or without it.

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
lead who answered — or one we've decided to drop — from being messaged again. **The only
ways a lead leaves the sequence are `reply` and `stop`.** Because there's no auto-exit,
`stop` is a deliberate, first-class action: it's how the BDR says "I'm done chasing this one."

***

## Touch vs Task — keep them distinct

| | **Touch** | **Task** |
| :-- | :-- | :-- |
| Tense | **Past** — "I reached out / followed up" | **Future** — "do X by this date" |
| Is | A logged *event* (history / audit) | A *reminder* that drives the work queue |
| Carries | touch #, date, channel?, optional detail | due date, text, assignee |
| Created | *after* the BDR does the outreach | *before* — it's the to-do |

They're two faces of one step: **recording a touch closes the reminder that prompted it and
schedules the next one.** The touch is the receipt; the task is the ticket.

**Single-writer rule — this is the important part:**

- **Cadence task** = the one "next-action" reminder for a lead in the sequence — the
  reach-out, then each follow-up. **Only the CLI** creates and closes it — on qualification
  (when a lead enters the reach-out status, the CLI opens the reach-out task) and on each
  `touch record`. There is **exactly one open per lead** at a time, tagged `[dogfu:cadence]`.
  **Never** hand-create or hand-complete a cadence task — letting the CLI be the single writer
  is the only way the outreach state stays in sync.
- **Ad-hoc task** = anything that isn't the next scheduled touch ("send the deck Friday",
  "check back after their raise closes", "intro to their CTO next week"). These are fine for
  you to create on the BDR's request via `task create`; they carry their own due date and
  **don't** touch outreach state. They're tagged distinctly from cadence tasks so the work
  queue and reports never confuse the two.

So: **the reach-out and every follow-up stay *touches* (events). The forward reminder is a
*task*. Discretionary one-offs are *ad-hoc tasks*. Nothing else becomes a task.**

***

## The work queue — one list, two kinds of action

The BDR's daily question is "who do I act on today?" **Run `crm touch due`** — it returns the
whole queue as **one list**, most-overdue first, each row tagged with its next action:

1. **Reach-outs** — Qualified leads with **no touch yet** (`touch_stage` null). "New people to
   say hello to."
2. **Follow-ups** — leads whose `next_touch_due ≤ today`. "People to chase," labelled with
   which follow-up is next and the channels already tried (so the BDR can vary channel).

**The model: the queue is a lead's open cadence task.** Every lead in the sequence carries
exactly one open cadence task standing for its next action — and **the reach-out is simply the
first cadence task** (opened when the lead enters the reach-out status), the follow-ups the ones
after it. So "who do I act on today?" is just **the open cadence tasks due today** — which is why
`crm touch due` and Close's native pending-task list are two views of the same queue.

> **Close's task list is a complete queue.** Because the reach-out, too, is a real cadence task,
> every item in the queue is a real Close task — so a BDR can work straight from Close's own
> Tasks/Inbox, and `crm touch reconcile` is the backstop that keeps that list trustworthy. Either
> way, **`crm touch due` is the command to run here.**

***

## Running `dogfu`

> **Where a flag's exact spelling depends on the CLI, follow `dogfu crm <cmd> --help`.**

`dogfu` is a published CLI (`pip install dogfu`) on your PATH — invoke it from anywhere, no
`uv run`, no mounted directory:

```bash
dogfu crm <command> [flags]
```

- **Output is JSON by default** — read stdout. Add `-o FILE` to dump a large list to a file
  and read back only what you need. `-f table` is for humans, not you.
- **Auth** is a `dogfu` session token via the **dogfu MCP**. If a command fails with an auth
  error (`missing dogfu token`, `invalid or expired token`) or `dogfu` isn't configured, call
  the MCP's **`get_setup_instructions`** tool and follow it: install the CLI, then
  `dogfu configure --otp <OTP> --title "Lead touch: <who/what>"`. Re-run and retry.
- **CRM (Close) calls** use the Close API key set once in the admin **Console → CRM
  Integration**. If it's missing, `crm` commands return a "no Close CRM API key configured" /
  `412` — surface that rather than retrying.
- Read calls are independent HTTP requests; run them concurrently if it helps.

***

## The command surface

Everything you need lives under `dogfu crm`. Popular flags are inline below so you rarely
need `--help`. All leaf commands take `-f json|table` (default json) and `-o FILE`.

### Reading state

| Goal | Command |
| :-- | :-- |
| List statuses + their ids (do this first; ids are account-specific) | `dogfu crm status list` |
| Who am I (the Close user) | `dogfu crm whoami` |
| List or find leads (no filter = newest first; narrow by name / query / status) | `dogfu crm lead list [-n "<name>"] [-q "<query>"] [-s <status_id>] [-l N] [--sort -date_created]` |
| Full lead incl. contacts + outreach fields | `dogfu crm lead get <lead_id>` |
| **The work queue** (reach-outs + follow-ups due now) | `dogfu crm touch due [-l N]` |
| **Outreach-load summary** (reach-outs vs follow-ups due + by-touch distribution) | `dogfu crm touch report` |
| A lead's **touch history** (each touch: #, date, channel, detail) | `dogfu crm touch history <lead_id> [-l N]` |
| A lead's notes / tasks / contacts | `dogfu crm note list <lead_id>` · `dogfu crm task list [-l <lead_id>] [-p]` · `dogfu crm contact list <lead_id>` |

A `Lead` from any of these carries `status_label`, the outreach fields (`touch_stage`,
`last_touched`, `next_touch_due`, `touch_channel` = channels tried), and embedded `contacts[]`
(with each person's `urls`). If a queue lead has no embedded contacts, `lead get <id>` to
fetch them so you can hand the BDR the LinkedIn/X handle to message.

### Recording outreach & moving leads through the sequence

| What happened | Command | Effect |
| :-- | :-- | :-- |
| Sent the **next** touch (reach-out or follow-up) | `dogfu crm touch record <lead_id> [-c <channel>] [--detail "<text>"]` | auto-advances to the lead's next touch; stamps `last_touched`=today, computes `next_touch_due`, **adds `<channel>` to the channels-tried set** (if given), appends a structured touch-history entry. **Closes the prior cadence task and opens the next.** Channel and detail are **optional**. |
| Sent a touch with a **non-default wait** | add `--wait-days N` | overrides the gap before the next follow-up is due |
| Recorded a touch but **don't** want the auto note/task | add `--no-note` and/or `--no-task` | suppresses the audit note / next-touch reminder |
| Lead **replied / positive** | `dogfu crm touch reply <lead_id>` | ends the sequence (clears `next_touch_due`) and moves the lead to **Engaged** automatically — the **handoff out of cold** (see below). Pass `--status <id>` only to override. |
| **Giving up** | `dogfu crm touch stop <lead_id> [--status <Bad Fit or Not Interested id>]` | same mechanism as reply; intent + status differ |
| Change **status only**, no sequence change | `dogfu crm lead update <lead_id> -s <status_id>` | funnel label only |

Notes on the verbs:
- `record` **auto-advances** from the lead's current touch — on a never-touched lead it logs
  the **reach-out**; otherwise the **next follow-up**. Don't force a number unless you mean to
  redo a touch, and don't record the same touch twice (check `touch_stage` if unsure).
- **Channel is freeform and optional.** Pass `-c linkedin|email|x|call|other` (or whatever the
  BDR used) when they tell you; omit it when they don't. It's added to the lead's
  channels-tried set and stamped on that touch's history entry.
- **Detail is optional.** Pass `--detail "<message or context>"` to store what they actually
  sent; omit when there's nothing to add.
- `record` defaults to **logging a note and creating the next-touch reminder task** — that's
  the BDR's automatic reminder and what `touch history` reads back. Leave them on unless asked.
- `record` does **not** change status (status stays a human label). `reply` moves the lead to
  **Engaged** on its own; pass `--status` on `reply`/`stop` (or `lead update -s`) only to set a
  *different* status.
- `reply` vs `stop`: both end the sequence and pull the lead from the follow-up queue. `reply`
  = they answered (→ **Engaged**, set automatically); `stop` = we're done (→ Bad Fit or Not
  Interested via `--status`). The difference is intent. Since nothing auto-ends the sequence,
  **`stop` is how a chase ends.**
- **A `reply` → Engaged is the handoff out of this skill.** Cold/`lead-touch` is done with the
  lead; the **`lead-engage`** skill works the live deal from there (discovery → opportunity →
  trial → proposal → won/lost). Don't keep recording touches on an Engaged lead — it has left
  the cadence. (A trial is a deal *stage* that `lead-engage` tracks as an opportunity, not a
  cold-outreach outcome — there's no Trial lead status.)

### Editing the records (leads, contacts, notes, tasks)

| Goal | Command |
| :-- | :-- |
| Create / update / delete a lead | `dogfu crm lead create -n "<name>" [-u <url>] [-d "<desc>"] [-s <status_id>]` · `lead update <lead_id> [-n] [-u] [-d] [-s]` · `lead delete <lead_id> [-y]` |
| Set curated company fields | on `lead create`/`update`: `--industry "<choice>"` (validated against Close's list) · `--employees <n>` · `--revenue <usd>` · `--business-model "<text>"` · `--seo-pages <n>` |
| Add / update / delete a contact (links go in `-u`, repeatable) | `dogfu crm contact create <lead_id> -n "<name>" [-t "<title>"] [-u <url>]... [-e <email>]... [-p <phone>]...` · `contact update <contact_id> [-n] [-t]` · `contact delete <contact_id> [-y]` |
| Add / list notes | `dogfu crm note create <lead_id> -t "<text>"` · `note list <lead_id> [-l N]` — note bodies are stored as plain text; don't rely on literal `<`/`>`/`&` for structure |
| Manage **ad-hoc** follow-up tasks (never the cadence task) | `dogfu crm task create -l <lead_id> -t "<text>" [-d YYYY-MM-DD] [-a <user_id>]` · `task list [-l <lead_id>] [-p]` · `task complete <task_id>` · `task update <task_id> [...]` · `task delete <task_id> [-y]` |

***

## Reporting the funnel / outreach load

Two parts: the **outreach load** has a native command; the **funnel by status** you compose.

1. **Outreach load due today — `crm touch report`.** It returns reach-outs-due vs follow-ups-due
   and the follow-up distribution by next touch number (Follow-up 1 / 2 / 3+), computed for you —
   don't re-derive it by hand. (`crm touch due` is the actionable list behind those counts.)
2. **Funnel by status** (no aggregate endpoint — compose it): `crm status list` → the ids, then
   per status `crm lead list -s <id> -l <high> -o file` and count → leads in each funnel status.

Lead with the **number that needs work today** (from `touch report`), then a funnel table
(status → count) and the outreach table (reach-outs due, follow-ups due). **Per-status counts are
capped by `--limit`** — for a big funnel, raise the limit or report "at least N" and say so.

***

## Operating rules

- **Resolve status ids first** with `crm status list`; map labels at runtime (fit/working →
  Qualified, replied → Engaged, dead → Bad Fit / Not Interested). Never
  hardcode an id.
- **Speak in actions, not stage math.** The queue already tells you each lead's next action
  ("Reach-out" or "Follow-up N") and the channels already tried — relay that. You don't make
  the BDR compute stage numbers.
- **Channel and detail are optional but valuable.** Log the channel whenever the BDR mentions
  it, and the message/detail when they share it — that's what lets them later see how each
  touch went and which channels are still worth trying. Never block on them.
- **Never assume a reply.** Close can't see LinkedIn/X replies. Only run `touch reply` when
  the BDR states the lead responded.
- **`stop` ends a chase.** Nothing auto-ends the sequence, so when the BDR is done with a lead,
  `touch stop` it (with the right `--status`) — don't just let it sit due forever.
- **Don't touch the cadence task by hand.** Let `record`/`reply`/`stop` manage the one
  system reminder. Create only **ad-hoc** tasks yourself, and only on request.
- **Don't double-record.** `record` advances from the current touch; running it twice skips
  one. Check `touch_stage` if unsure.
- **Put data where it belongs:** a person's LinkedIn/X in that contact's `-u` urls; the
  message sent on a touch via `--detail`; reusable context in a note; reminders as ad-hoc
  tasks; the brief headline in the lead `description`.

## Response format

- **Work queue:** a numbered / tabular list — lead name + company, the contact(s) with their
  LinkedIn/X links, last touched, days overdue, **channels already tried**, and the **next
  action** (Reach-out, or Follow-up N — suggest a fresh channel if useful). Make it
  copy-paste actionable so the BDR can go message immediately.
- **After a write:** one line confirming what changed — lead, the touch just logged (with
  channel if given), and the next due date (or "sequence ended").
- **Status/report:** the headline "needs work today" number first, then the funnel + outreach
  tables.
