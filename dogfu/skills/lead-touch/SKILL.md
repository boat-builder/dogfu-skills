---
name: lead-touch
description: >-
  Work Agent Berlin's leads in the Close CRM — everything after research, from cold outreach
  through a closed deal — using the `dogfu` CLI. Use this whenever someone wants to interact
  with the CRM about leads that already exist, in either phase. Cold cadence: check a lead's
  status or where it stands in the outreach sequence, pull what's due and record the touches
  they just sent (a reach-out or a follow-up, on any channel), mark a lead as replied / not
  interested / nurture, change a lead's status, read a lead's touch history, add a note or
  contact or follow-up task, or report on the funnel / outreach load. Live deals: run a
  discovery / intro call, open or advance or close an **opportunity** (Discovery → Trial →
  Proposal → Won/Lost), log a call / demo / proposal that just happened, set or check the
  **next step** on a deal, pull the deals that need action today or have stalled, mark a deal
  won or lost, intake an inbound lead, or report the deal pipeline / forecast. The BDR does
  the real-world outreach and calls themselves; this skill is how they tell the CRM what
  happened and ask it what to do next. For the cross-phase worklist of everything to work
  today ("what should I work on", any or all stages, read-only, with context), run
  `dogfu crm worklist`; for researching or qualifying a brand-new target, use the
  lead-research skill — this is the layer that runs after that.
---

# Lead touch — work leads in Close, cold cadence through closed deal (powered by `dogfu`)

This skill is how a BDR **talks to the CRM about leads that are already in it** — across the
whole post-research lifecycle. The BDR does the real-world work by hand — sends the LinkedIn
request, the DM, the email; takes the discovery call; runs the demo. Your job is to translate
what they tell you into the right `dogfu crm` calls: read where a lead stands, pull what's
due, record what happened, move statuses and deal stages, and keep notes/tasks/contacts
straight. The `dogfu` CLI holds the Close credentials and commands; you hold the model of how
our leads are organized — below, plus the flow reference you load — so you rarely need
`--help`.

The work here is open-ended — far more than a fixed list of plays. Anything a BDR would do in
Close, they can ask you to do. Reason from the model rather than pattern-matching to a script.

***

## Two flows — load the reference for the one in play

The lifecycle has a cold half and a warm half, and they run on different machines: cold runs
on a **cadence** (a system-computed next touch), warm runs on **next steps** (one explicit
committed action per deal) and **opportunity stages**. Before acting, read the reference for
the flow the request touches:

- **Cold outreach — the cadence** → read **`references/cold-outreach.md`**. Leads being
  *chased*: pulling reach-outs and follow-ups due, recording touches the BDR sent, marking a
  reply, stopping a chase, touch history, the funnel / outreach-load report.
- **Live deals — the warm phase** → read **`references/deals.md`**. Leads that *answered* (or
  came in inbound): landing the discovery call, opening / advancing / closing an
  **opportunity** (Discovery → Trial → Proposal → Won/Lost), setting next steps, inbound
  intake, the pipeline / forecast report.

Read **both** when the ask spans the seam — a reply just came in (a cold verb whose outcome
starts the warm phase), or a report covering funnel *and* pipeline. Everything below is the
shared foundation both flows assume.

***

## How our leads are organized in Close

A **lead is a company** (not a person). Each lead carries:

- **Identity** — `name`, `url` (website), `description` (a short headline).
- **Status** (`status_id` + `status_label`) — where it sits in the **funnel**. A human
  judgment label, **account-specific** — resolve the live ids with `crm status list`, never
  hardcode them. Typical labels: *Potential, Qualified, Connected, Engaged, Customer, Bad
  Fit, Not Interested, Canceled, DNC*.
- **Contacts** — the people you actually reach out to. Each has `urls` (their LinkedIn / X —
  what you DM from), `emails`, `phones`, `title`.
- **Outreach state** — system-managed fields tracking the cold sequence (cold reference).
- **Opportunities** — the actual deals you forecast, zero or more per lead, each with a
  pipeline stage, value, deal type, and confidence (deals reference).
- **Curated company fields** — `industry`, `employees`, `revenue`, `business_model`,
  `seo_pages`. Set by lead-research; you rarely touch them here.
- Attached **notes** (write-ups, audit trail) and **tasks** (reminders that drive the queue).

**Three layers — don't conflate them.** *Status* is the funnel position (a human lifecycle
label). *Outreach state* is the cold-sequence position (machine-tracked; touches never change
status). *Opportunity* is the deal itself (the thing you forecast; a lead's status mirrors
whether one is open). Each flow reference covers its layer's mechanics.

**The lifecycle in one line:**

```
Qualified ──cold cadence: reach-out → follow-ups──▶ reply → Connected ──discovery call
confirms a deal (the gate)──▶ Engaged (opportunity: Discovery → Trial → Proposal) ──won──▶
Customer   (exits anywhere → Bad Fit / Not Interested / Canceled / DNC)
```

The **reply → Connected** move is the seam between the flows: the cold cadence ends and the
warm phase (land the discovery call, then the gate) begins.

***

## The task discipline — CLI-owned reminders, single writer

Every stage of the lifecycle drives the work queue through **exactly one open, CLI-owned
next-action task per unit of work**, each with its tag. **The CLI is the single writer of all
three — never hand-create or hand-complete a tagged task**; letting the CLI own them is the
only way the state stays in sync:

- **`[dogfu:cadence]`** — the cold next-touch reminder, one open per lead in the sequence.
  Opened when a lead enters the reach-out status; swapped by each `touch record`; closed by
  `reply` / `stop` / leaving the cold flow.
- **`[dogfu:engage]`** — the pre-gate task ("Land the discovery call"), one open per
  Connected lead. Opened the moment a lead becomes Connected; retired when the lead leaves
  Connected (a deal opens, or it goes terminal).
- **`[dogfu:deal:<opp_id>]`** — the next-step task, one open per open opportunity. Its text
  and due date are a *human decision*, but only the `opportunity` verbs write it.

**Ad-hoc tasks** are the exception: anything that isn't a system next-action ("send the deck
Friday", "intro to their CTO next week"). Those are fine for you to create on the BDR's
request via `task create` — they carry no dogfu tag and never touch system state.

**`dogfu crm reconcile`** is the backstop that audits these invariants across the whole book
(missing task, stray task, zombie state) and can auto-repair them — the read-only audit pass
around it is the **crm-cleanup** skill.

***

## The work queue — one command, five kinds

**`dogfu crm worklist`** reads the BDR's actual due-task inbox (every open task due ≤ today)
and classifies every row with a `kind` and a human `next_action`: `reach-out` and `follow-up`
(cold — cold reference), `engage` and `deal` (warm — deals reference), and `ad-hoc` (the
task's own text). `--kind <k>` slices it; plain `worklist` is the whole queue,
most-overdue-first. Because every system next-action is a real Close task, this and Close's
native Tasks/Inbox are literally the same list.

For the *cross-phase* daily brief ("what should I work on across everything", read-only,
most-overdue-first), run plain **`dogfu crm worklist`** with no `--kind`; in here you pull the
slice the flow you're working needs.

***

## Running `dogfu`

> **Where a flag's exact spelling depends on the CLI, follow `dogfu crm <cmd> --help`.**

`dogfu` is a published CLI (`pip install dogfu`) on your PATH — invoke it from anywhere, no
`uv run`, no mounted directory:

```bash
dogfu crm <command> [flags]
```

- **First, authenticate the CLI.** Before any `dogfu` command, call the dogfu MCP's
  **`get_setup_instructions`** tool and follow it: install the CLI (if needed), then
  `dogfu configure --otp <OTP> --title "Lead touch: <who/what>"` with the OTP it returns.
  This is the first step whenever the skill runs.
- **Output is JSON by default** — read stdout. Add `-o FILE` to dump a large list to a file
  and read back only what you need. `-f table` is for humans, not you.
- **CRM (Close) calls** use the Close API key set once in the admin **Console → CRM
  Integration**. If it's missing, `crm` commands return a "no Close CRM API key configured" /
  `412` — surface that rather than retrying.
- Read calls are independent HTTP requests; run them concurrently if it helps.

***

## The shared command surface

Everything lives under `dogfu crm`. All leaf commands take `-f json|table` (default json) and
`-o FILE`. The flow-specific verbs (`touch …`, `opportunity …`) and their queue/report reads
are in the flow references; these are the shared reads and record-editing commands both flows
use.

### Reading state

| Goal | Command |
| :-- | :-- |
| List statuses + their ids (do this first; ids are account-specific) | `dogfu crm status list` |
| Who am I (the Close user) | `dogfu crm whoami` |
| List or find leads (no filter = newest first; narrow by name / query / status) | `dogfu crm lead list [-n "<name>"] [-q "<query>"] [-s <status_id>] [-l N] [--sort -date_created]` |
| Full lead incl. contacts + outreach fields | `dogfu crm lead get <lead_id>` |
| **The work queue** (slice by kind per flow) | `dogfu crm worklist [--kind reach-out\|follow-up\|engage\|deal\|ad-hoc] [-l N]` |
| A lead's notes / tasks / contacts | `dogfu crm note list <lead_id>` · `dogfu crm task list [-l <lead_id>] [-p]` · `dogfu crm contact list <lead_id>` |

A `Lead` from any of these carries `status_label`, the outreach fields, and embedded
`contacts[]` (with each person's `urls`). If a queue lead has no embedded contacts,
`lead get <id>` to fetch them so you can hand the BDR the LinkedIn/X handle to message.

### Editing the records (leads, contacts, notes, tasks)

| Goal | Command |
| :-- | :-- |
| Create / update / delete a lead | `dogfu crm lead create -n "<name>" [-u <url>] [-d "<desc>"] [-s <status_id>]` · `lead update <lead_id> [-n] [-u] [-d] [-s]` · `lead delete <lead_id> [-y]` |
| Set curated company fields | on `lead create`/`update`: `--industry "<choice>"` (validated against Close's list) · `--employees <n>` · `--revenue <usd>` · `--business-model "<text>"` · `--seo-pages <n>` |
| Add / update / delete a contact (links go in `-u`, repeatable) | `dogfu crm contact create <lead_id> -n "<name>" [-t "<title>"] [-u <url>]... [-e <email>]... [-p <phone>]...` · `contact update <contact_id> [-n] [-t]` · `contact delete <contact_id> [-y]` |
| Add / list notes | `dogfu crm note create <lead_id> -t "<text>"` · `note list <lead_id> [-l N]` — note bodies are stored as plain text; don't rely on literal `<`/`>`/`&` for structure |
| Manage **ad-hoc** tasks (never a dogfu-tagged task) | `dogfu crm task create -l <lead_id> -t "<text>" [-d YYYY-MM-DD] [-a <user_id>]` · `task list [-l <lead_id>] [-p]` · `task complete <task_id>` · `task update <task_id> [...]` · `task delete <task_id> [-y]` |

***

## Shared operating rules

- **Resolve ids at runtime.** `crm status list` for statuses (map labels: fit/working →
  Qualified, replied → Connected, deal open → Engaged, dead → Bad Fit / Not Interested);
  `crm opportunity pipelines` for deal stages. Never hardcode an id.
- **Never assume an event.** Close can't see LinkedIn/X/email/calls. Only record a touch, a
  reply, a call, a trial, or a signature when the BDR states it happened.
- **Respect the single-writer rule.** The three dogfu-tagged tasks belong to the CLI; you
  create only ad-hoc tasks, and only on request.
- **Speak in actions, not internals.** The queue already labels each lead's `next_action` —
  relay that; don't make the BDR compute touch numbers or stage math.
- **Put data where it belongs:** a person's LinkedIn/X in that contact's `-u` urls; what
  happened in a touch's `--detail` or a note; deal economics on the opportunity; reusable
  context in a note; reminders as ad-hoc tasks; the brief headline in the lead `description`.
- **After a write:** confirm in one line what changed and what's next (each flow reference
  has the exact format for its queues and reports).
