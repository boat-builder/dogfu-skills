---
name: lead-worklist
description: >-
  Pull the single, prioritized list of leads a BDR should work right now from Agent Berlin's
  Close CRM — read-only, across the whole lifecycle — using the `dogfu` CLI. Use this whenever
  someone wants their **worklist / work queue / daily plan**: "who do I work on today", "what's on
  my plate", "give me the next N leads", "show me everyone due", or a narrower slice — "just my
  reach-outs", "follow-ups due", "leads I'm engaging", "deals that need action" — in **one stage or
  across all stages**, each row carrying the contact and the context to act. It composes the cold
  queue (reach-outs + follow-ups), the warm deal queue, and pre-gate engaged leads into one ranked
  list and hands each row off to the skill that records the action. It is **read-only — it never
  writes to the CRM**: to record a touch you sent use **lead-touch**, to move a deal use
  **lead-engage**, for a health / anomaly audit use **crm-cleanup**, and to research a brand-new
  target use **lead-research**.
---

# Lead worklist — the read-only "what do I work on now" queue (powered by `dogfu`)

This skill answers the BDR's first question every morning — **"who should I work on, and what do I
do next?"** — across the entire lifecycle in one list. It is **read-only**: it composes `dogfu crm`
*read* calls into a single prioritized worklist, attaches the context to act, and points each row at
the skill that records the action. It never changes CRM state.

It sits **across** the phase skills, not inside them: **lead-touch** runs the cold cadence and
**lead-engage** works live deals — each has its own *in-session* queue for the loop it owns. This
skill is the **cross-phase daily brief** that unifies them, so "what's on my plate" has one home.
The depth of each model lives in those skills; this one assumes them and only recaps what assembling
the list needs.

***

## The lifecycle, and where each row comes from

A lead moves research → cold outreach → engaged → live deal. The worklist is the union of every
stage that has an action due, pulled from the purpose-built queue commands. **Every item is a real
Close task** (the reach-out task is opened on qualification; follow-ups and deal next-steps are CLI-
owned too) — so this is your *task list*, not a guess, and it's exactly "only what's due", never
random.

| Stage (what to do) | Source command | Phase |
| :-- | :-- | :-- |
| **Reach-out** — Qualified, not yet contacted | `dogfu crm touch due` (the never-touched rows) | cold |
| **Follow-up N** — next cadence touch due | `dogfu crm touch due [--stage N]` | cold |
| **Engaging (pre-gate)** — replied / inbound, no opportunity yet | `dogfu crm lead list -s <Engaged id>` + its open ad-hoc task | warm |
| **Live deal** — open opportunity due / dropped / stalled | `dogfu crm opportunity due` | warm |

- **All stages** = run the cold queue **and** the warm queue (**and** pre-gate Engaged) and merge
  into one list, most-urgent first.
- **One stage** = run just the matching source — only reach-outs, `--stage 1` for follow-up 1, only
  `opportunity due` for deals, etc.
- **N leads** = pass `-l N` to each source and cap the merged list to what they asked for.

> **Reach-outs are real tasks now.** A lead gets its reach-out cadence task the moment it's
> qualified, so the cold queue already includes never-touched leads — you don't reconstruct them
> from status. Close's own Tasks/Inbox is the same queue; this skill is the **cross-phase** view of
> it, with the warm deals folded in.

One case the queue commands can't catch on their own: a **pre-gate Engaged lead** (it replied or came
inbound but has no opportunity yet) is invisible to `opportunity due`. Pull those from the Engaged
lead list so a conversation that stalls *before* a deal exists doesn't slip.

***

## Context — tier it; don't summarize the whole queue, and don't bury it in a file

A worklist is only useful if each row says *who to contact and what to do next*. That context comes
in two layers — keep them separate and the list stays both cheap and actionable.

**Layer 1 — act-on-it context, inline, for every row.** The queue commands already embed it, so
present it straight from their payload — **don't re-read and don't re-summarize**:

- the **contact** to reach — name, title, and the **LinkedIn / X handle to DM** (from the lead's
  `contacts[]`; if a row has no contact embedded, one `lead get <id>` fills it),
- the **next action** — Reach-out, Follow-up N, or the deal's stage + next step,
- **channels already tried** (`touch_channel`) so they can vary channel,
- **last touched / days overdue**, and the lead's one-line **headline** (`description`).

This is what makes the list copy-paste actionable, and it costs **one queue call for the whole list**
— not one read per lead — so a 50-row worklist stays cheap.

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

- **Output is JSON by default** — read stdout. `-o FILE` dumps a large queue to a file; `-f table`
  is for humans, not you.
- **Auth** is a `dogfu` session token via the **dogfu MCP**. On an auth error (`missing dogfu
  token`, `invalid or expired token`), call the MCP's **`get_setup_instructions`** and follow it
  (`dogfu configure --otp <OTP> --title "Lead worklist"`), then retry.
- **CRM (Close) calls** use the Close API key set in the admin **Console → CRM Integration**; a
  missing key returns a `412` — surface it, don't retry.
- Every call here is an independent **read** — **run them concurrently** (the cold queue, the warm
  queue, and the Engaged list have no ordering between them).

***

## The command surface (all reads)

| Goal | Command |
| :-- | :-- |
| List statuses + their ids (do this first; ids are account-specific) | `dogfu crm status list` |
| **Cold queue** — reach-outs + follow-ups due now (filter one stage with `--stage N`) | `dogfu crm touch due [--stage N] [-l N]` |
| **Warm queue** — deals due / dropped / stalled | `dogfu crm opportunity due [-l N]` |
| **Pre-gate Engaged leads** — replied/inbound, no opp yet | `dogfu crm lead list -s <Engaged id> [-l N]` |
| Full lead incl. contacts + outreach fields (fills a row with no embedded contact) | `dogfu crm lead get <lead_id>` |
| Deep context for a lead being actioned | `dogfu crm note list <lead_id>` · `dogfu crm touch history <lead_id>` |
| Outreach-load summary (counts behind the cold queue) | `dogfu crm touch report` |

***

## Operating rules

- **Read-only — never write.** This skill assembles and presents; it does **not** record touches,
  move statuses, open or advance deals, or create tasks. End each row with the command/skill that
  *does*: **lead-touch** (`touch record` / `reply` / `stop`) for cold rows, **lead-engage**
  (`opportunity …`) for deals. Hand off; don't act.
- **Resolve status ids first** (`crm status list`) before filtering by Engaged or any status. Never
  hardcode an id.
- **Tier the context** (above): Layer 1 inline for all rows; Layer 2 only for the lead being worked.
  Don't fan out deep reads across the whole queue.
- **Speak in actions, not stage math.** The queue already labels each row's next action and the
  channels tried — relay that; don't make the BDR compute touch numbers.
- **Respect limits.** Each source is capped by `-l`; if you cap a stage, say "at least N" rather than
  implying the list is complete.
- **Present facts, don't fabricate.** If a field is missing (no contact link, no research note), say
  so and point at the fix (e.g. lead-research to enrich, or crm-cleanup if it looks like drift) —
  don't invent a hook or a handle.

***

## Response format

- **Worklist:** a numbered / tabular list, most-urgent first (or grouped by stage when they asked for
  a slice). Each row: lead + company, the **contact(s) with LinkedIn/X links**, the **next action**
  (Reach-out / Follow-up N / deal stage + next step), **channels already tried**, last touched / days
  overdue, and the **command to act** (which skill + verb records it). Make it copy-paste actionable
  so the BDR can go message immediately.
- **Drill-in (one lead):** the Layer-2 brief — research / hooks, prior touches and what was said,
  then a suggested next angle + fresh channel. This is the **only** place you synthesize.
- **Empty queue:** say it plainly ("nothing due today") rather than padding the list.
