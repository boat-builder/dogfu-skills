---
name: lead-worklist
description: >-
  Pull the single, prioritized list of leads a BDR should work right now from Agent Berlin's
  Close CRM — read-only, across the whole lifecycle — using the `dogfu` CLI. Use this whenever
  someone wants their **worklist / work queue / daily plan**: "who do I work on today", "what's on
  my plate", "give me the next N leads", "show me everyone due", or a narrower slice — "just my
  reach-outs", "follow-ups due", "discovery calls to land", "deals that need action", "my ad-hoc
  tasks" — in **one kind or across all**, each row carrying the contact and the context to act. It
  runs `dogfu crm worklist`, which reads the real due-task inbox and classifies every row (reach-out
  / follow-up / engage / deal / ad-hoc), then hands each off to the skill that records the action.
  It is **read-only — it never
  writes to the CRM**: to record a touch you sent or to move a deal use **lead-touch**, for a
  health / anomaly audit use **crm-cleanup**, and to research a brand-new target use
  **lead-research**.
---

# Lead worklist — the read-only "what do I work on now" queue (powered by `dogfu`)

This skill answers the BDR's first question every morning — **"who should I work on, and what do I
do next?"** — across the entire lifecycle in one list. It is **read-only**: it runs `dogfu crm
worklist` (the CLI does the composing — reading the actual due-task inbox and classifying every row),
presents it with the context to act, and points each row at the skill that records the action. It
never changes CRM state.

It sits **across** the phases, not inside them: **lead-touch** owns the actions — its cold flow
runs the outreach cadence, its warm flow works live deals. This skill is the **cross-phase daily
brief** that unifies them, so "what's on my plate" has one home. The depth of each model lives in
that skill's flow references; this one assumes them and only recaps what presenting the list needs.

***

## Where the worklist comes from — one command

A lead moves research → cold outreach → connected → live deal. **`dogfu crm worklist` returns the
whole cross-phase queue in one call.** It reads the BDR's actual Close inbox — every open task due today or
earlier — and classifies each row by what kind of work it is, enriched with the context to act. This
*is* the task list, not a reconstruction from lead state: nothing due is invisible, and Close's own
Tasks/Inbox is literally the same list.

Each row carries a **`kind`** and the human **`next_action`**:

| `kind` | What it is | `next_action` |
| :-- | :-- | :-- |
| `reach-out` | Qualified lead, never contacted | "Reach-out" |
| `follow-up` | next cadence touch due | "Follow-up N" |
| `engage` | pre-gate **Connected** lead (replied, no deal yet) | "Land the discovery call" |
| `deal` | an open opportunity's next step | the deal's next step (+ `opportunity_stage`) |
| `ad-hoc` | a manual task, outside the cadence/engage/deal systems | the task's own text |

- **All stages** (default): `dogfu crm worklist` — the whole inbox, most-overdue-first.
- **One slice**: `dogfu crm worklist --kind reach-out|follow-up|engage|deal|ad-hoc`. (`--kind
  follow-up` gives every follow-up; to isolate "follow-up 1", filter the rows by `touch_stage`
  client-side.)
- **N rows**: `-l N`. **One person's inbox**: `--mine` (or `--assignee <user_id>`); the default shows
  every assignee's due tasks, so nothing hides behind an assignee filter.

> **Ad-hoc tasks count.** A manual "Reach out to <name>" task with no dogfu tag is still real, due
> work — it surfaces as an `ad-hoc` row.

What `worklist` deliberately does **not** show: a lead or deal whose *state* says it needs action but
that has **no task** — a Qualified lead missing its reach-out task, a Connected lead missing its
engage task, an open opportunity with no next step. Those aren't in the inbox; they're **drift**,
surfaced (and repaired) by `dogfu crm reconcile` via the **crm-cleanup** skill. So the split is clean:
**`worklist` = what's actually due now; `reconcile` = what *should* have a task but doesn't.**

***

## Context — tier it; don't summarize the whole queue, and don't bury it in a file

A worklist is only useful if each row says *who to contact and what to do next*. That context comes
in two layers — keep them separate and the list stays both cheap and actionable.

**Layer 1 — act-on-it context, inline, for every row.** `crm worklist` embeds it on each row, so
present it straight from the payload — **don't re-read and don't re-summarize**:

- the **contact(s)** to reach — name, title, and the **LinkedIn / X handle to DM** (each row's
  `contacts[]`, already embedded — no `lead get` needed),
- the **`kind`** and **`next_action`** — Reach-out, Follow-up N, the deal's next step (with
  `opportunity_stage`), or the ad-hoc task's own text,
- **channels already tried / still untried** (`channels_tried` / `channels_untried`) so they can
  vary channel,
- **`days_overdue`**, **`last_touched`**, and the lead's **`status_label`** for triage.

This is what makes the list copy-paste actionable, and it costs **one `worklist` call for the whole
list** — not one read per lead — so a 50-row worklist stays cheap.

**Layer 2 — deep context, on demand, only for the lead being actioned.** The full research write-up,
DM hooks, prior notes, and what was said in past touches are large and only matter when the BDR sits
down to message *one* lead. Pull them then, for that lead only:

- `dogfu crm lead get <id>` — curated company fields + all contacts,
- `dogfu crm note list <id>` — the research write-up, DM hooks, history,
- `dogfu crm touch history <id>` — what each prior touch said and on which channel.

**You are not forced to read Layer 2 for the whole queue** — that's the point of the split. Read it
only when the BDR drills into a lead, and *there* it is worth adding judgment: from the hooks +
history, suggest the next angle and a fresh channel. **At the list level, present facts; at the point
of action, advise.**

***

## Running `dogfu`

> **Where a flag's exact spelling depends on the CLI, follow `dogfu crm <cmd> --help`.**

Same dependency and auth as the other skills, and **read-only throughout**. `dogfu` is a published
CLI (`pip install dogfu`) on your PATH:

```bash
dogfu crm <command> [flags]
```

- **First, authenticate the CLI.** Before any `dogfu` command, call the dogfu MCP's
  **`get_setup_instructions`** tool and follow it: install the CLI (if needed), then
  `dogfu configure --otp <OTP> --title "Lead worklist"` with the OTP it returns. This is the
  first step whenever the skill runs.
- **Output is JSON by default** — read stdout. `-o FILE` dumps a large queue to a file; `-f table`
  is for humans, not you.
- **CRM (Close) calls** use the Close API key set in the admin **Console → CRM Integration**; a
  missing key returns a `412` — surface it, don't retry.
- Every call here is an independent **read** — the worklist is one call; any extra (the Engaged list,
  a drill-in) can **run concurrently**.

***

## The command surface (all reads)

| Goal | Command |
| :-- | :-- |
| **The worklist** — the whole due inbox, classified + enriched (this is the main call) | `dogfu crm worklist [--kind reach-out\|follow-up\|engage\|deal\|ad-hoc] [--mine] [-l N]` |
| List statuses + their ids (only if you need to pull a status list below) | `dogfu crm status list` |
| **Connected / Engaged leads** — the warm book, if you want it by status (pre-gate work is already in the worklist as `engage` rows) | `dogfu crm lead list -s <Connected\|Engaged id> [-l N]` |
| Full lead incl. contacts + outreach fields (worklist rows already embed contacts; use for the rare gap) | `dogfu crm lead get <lead_id>` |
| Deep context for a lead being actioned | `dogfu crm note list <lead_id>` · `dogfu crm touch history <lead_id>` |
| Cold-outreach load summary (counts, agrees with the worklist) | `dogfu crm touch report` |
| What's **drifted** (owed but no task) — hand to the crm-cleanup skill | `dogfu crm reconcile` |

***

## Operating rules

- **Read-only — never write.** This skill assembles and presents; it does **not** record touches,
  move statuses, open or advance deals, or create tasks. End each row with the command/skill that
  *does* — **lead-touch** in the matching flow: `touch record` / `reply` / `stop` for cold
  `reach-out`/`follow-up` rows; the deals flow for `engage` rows (land the discovery call → open
  the opportunity) and `deal` rows (`opportunity …`). Hand off; don't act.
- **Resolve status ids first** (`crm status list`) before filtering by Engaged or any status. Never
  hardcode an id.
- **Tier the context** (above): Layer 1 inline for all rows; Layer 2 only for the lead being worked.
  Don't fan out deep reads across the whole queue.
- **Speak in actions, not stage math.** Each row already carries its `next_action` and channels
  tried — relay that; don't make the BDR compute touch numbers.
- **Respect limits.** `worklist` is capped by `-l`; if you cap it, say "at least N" rather than
  implying the list is complete.
- **Present facts, don't fabricate.** If a field is missing (no contact link, no research note), say
  so and point at the fix (e.g. lead-research to enrich, or crm-cleanup if it looks like drift) —
  don't invent a hook or a handle.

***

## Response format

- **Worklist:** a numbered / tabular list, most-urgent first (or grouped by stage when they asked for
  a slice). Each row: lead + company, the **contact(s) with LinkedIn/X links**, the **next action**
  (Reach-out / Follow-up N / Land the discovery call / deal stage + next step), **channels already
  tried**, last touched / days
  overdue, and the **command to act** (which skill + verb records it). Make it copy-paste actionable
  so the BDR can go message immediately.
- **Drill-in (one lead):** the Layer-2 brief — research / hooks, prior touches and what was said,
  then a suggested next angle + fresh channel. This is the **only** place you synthesize.
- **Empty queue:** say it plainly ("nothing due today") rather than padding the list.
