---
name: lead-touch
description: >-
  Manage Agent Berlin's leads in the Close CRM after the BDR has done the actual
  outreach — using the `dogfu` CLI. Use this whenever someone wants to interact with
  the CRM about leads that already exist: check a lead's status or where it stands in
  the outreach cadence, pull the leads due for a follow-up ("who do I message today",
  "my work queue"), record a touch they just sent, mark a lead as replied / engaged /
  not interested / nurture, change a lead's status, read a lead's touch history, add a
  note or contact or follow-up task to a lead, or report on the funnel / cadence load.
  The BDR sends the messages themselves; this skill is how they tell the CRM what
  happened and ask it what to do next. For researching or qualifying a brand-new
  target, use the lead-research skill instead — this is the layer that runs after that.
---

# Lead touch — CRM & outreach-cadence ops (powered by `dogfu`)

This skill is how a BDR **talks to the CRM about leads that are already in it**. The BDR
does the real-world outreach by hand — sends the LinkedIn request, the DM, the email. Your
job is to translate what they tell you into the right `dogfu crm` calls: read where a lead
stands, pull what's due, record what they sent, move a lead's status, and keep notes/tasks/
contacts straight. The `dogfu` CLI holds the Close credentials and commands; you hold the
model of how our leads are organized, below — so you rarely need `--help`.

The work here is open-ended — far more than a fixed list of plays. Anything a BDR would do
in Close, they can ask you to do. The two sections that follow ("How our leads are
organized" and "The command surface") are the complete mental model and command set; reason
from them rather than pattern-matching to a script.

***

## How our leads are organized in Close

A **lead is a company** (not a person). Each lead carries:

- **Identity** — `name`, `url` (website), `description` (a short headline).
- **Status** (`status_id` + `status_label`) — where it sits in the **funnel**. A human
  judgment label, **account-specific** — resolve the live ids with `crm status list`, never
  hardcode them. Typical labels: *Potential, Qualified, Engaged, Bad Fit, Nurture*.
- **Contacts** — the people you actually reach out to. Each has `urls` (their LinkedIn / X —
  what you DM from), `emails`, `phones`, `title`.
- **Cadence state** — four system-managed fields tracking the outreach sequence (below).
- **Curated company fields** — `industry`, `employees`, `revenue`, `business_model`,
  `seo_pages`. Set by lead-research; you rarely touch them here.
- Attached **notes** (write-ups, audit trail) and **tasks** (follow-up reminders).

**Two layers — don't conflate them.** *Status* is the funnel position (a human label).
*Cadence state* is the outreach-sequence position (machine-tracked). They move
independently: recording a touch advances the cadence but **does not** change status;
changing status doesn't touch the cadence. Only `touch reply` / `touch stop` deliberately do
both (clear the cadence **and** optionally set a status).

### The outreach cadence

A lead is worked as a sequence of **touches** across channels, with a wait between each.
`dogfu` applies this schedule automatically (it lives in the CLI's `dogfu.cadence` config):

| Touch (stage) | Channel | What the BDR does | Wait before next |
| :-- | :-- | :-- | :-- |
| **0** | LinkedIn | connection request | ~3 days |
| **1** | LinkedIn | first DM | ~4 days |
| **2** | Email | email | ~6 days |
| **3** | X | DM / final touch | — (last; then exits) |

State lives in **four system-managed fields** on the lead — you never set these by hand; the
`touch` verbs compute them:

- **`touch_stage`** — index of the **last completed** touch (0-based). `null` = not yet in the cadence.
- **`last_touched`** — date the most recent touch went out.
- **`next_touch_due`** — date the next touch is due. **Empty = the lead has exited the cadence** (replied, stopped, or finished the final touch) and drops out of every work queue.
- **`touch_channel`** — channel of the last touch (`linkedin` | `email` | `x` | `other`).

**The "due" rule** (the core of the work queue): a lead is due when `next_touch_due ≤ today`.
`crm touch due` returns exactly those, most-overdue first. `--stage N` narrows to leads whose
**last completed** touch was N — i.e. the ones now due for touch **N+1**.

**Lifecycle:** `not in cadence → touch 0 → 1 → 2 → 3 → exits`. A **reply** at any point exits
the lead (clears `next_touch_due`), which is what stops a lead who answered from being
messaged again.

### Two gotchas that cause most mistakes

1. **"second touch" ≠ `--stage 2`.** The *second touch* is touch index **2** (the email),
   which you pull with **`--stage 1`** (leads that completed touch 1, now due for touch 2).
   Map the BDR's words to the stage *they just finished*, not the touch number.
2. **Touch 0 isn't in `touch due`.** Leads awaiting their first connection request aren't in
   the cadence yet (`next_touch_due` is empty), so `touch due` won't list them. Pull fresh
   leads to *start* outreach by **status** (`crm lead list -s <Qualified id>`), then
   `touch record` the connection request. When "first touch" is ambiguous — *start outreach*
   (touch 0) vs *send the first message* (touch 1, `--stage 0`) — ask one quick question.

**Reply detection is manual.** Close can't see LinkedIn/X replies. Only run `touch reply`
when the BDR tells you a lead responded. **Never assume a reply.**

***

## Running `dogfu`

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
- Read calls are independent HTTP requests; run them concurrently if it helps. Keep
  **mutations deliberate** — one lead at a time, after confirming (see Operating rules).

***

## The command surface

Everything you need lives under `dogfu crm`. Popular flags are inline below so you rarely
need `--help`. All leaf commands take `-f json|table` (default json) and `-o FILE`.

### Reading state

| Goal | Command |
| :-- | :-- |
| List statuses + their ids (do this first; ids are account-specific) | `dogfu crm status list` |
| Who am I (the Close user) | `dogfu crm whoami` |
| List leads (newest first, filter by status) | `dogfu crm lead list [-l N] [-s <status_id>] [--sort -date_created]` |
| Find a lead by name / raw query / status | `dogfu crm lead search [-n "<name>"] [-q "<query>"] [-s <status_id>] [-l N]` |
| Full lead incl. contacts + cadence fields | `dogfu crm lead get <lead_id>` |
| Leads **due for a follow-up now** | `dogfu crm touch due [--stage N] [-l N]` |
| A lead's **touch history** (the cadence notes) | `dogfu crm touch history <lead_id> [-l N]` |
| A lead's notes / tasks / contacts | `dogfu crm note list <lead_id>` · `dogfu crm task list [-l <lead_id>] [-p]` · `dogfu crm contact list <lead_id>` |

A `Lead` from any of these carries `status_label`, the four cadence fields (`touch_stage`,
`last_touched`, `next_touch_due`, `touch_channel`), and embedded `contacts[]` (with each
person's `urls`). If a queue lead has no embedded contacts, `lead get <id>` to fetch them so
you can hand the BDR the LinkedIn/X handle to message.

### Recording outreach & moving leads through the cadence

| What happened | Command | Effect |
| :-- | :-- | :-- |
| Sent the **next** touch | `dogfu crm touch record <lead_id>` | auto-advances to the lead's next stage; stamps `last_touched`=today, computes `next_touch_due` from the cadence wait, sets the channel. **Also logs a note and creates a follow-up task** due on the next touch. |
| Sent a **specific / non-default** touch | `dogfu crm touch record <lead_id> --stage N [-c linkedin\|email\|x\|other] [--wait-days N]` | as above, for stage N, with channel/wait overridden |
| Sent a touch but **don't** want the auto note/task | add `--no-note` and/or `--no-task` | suppresses the audit note / follow-up task |
| Lead **replied / positive** | `dogfu crm touch reply <lead_id> [--status <Engaged id>]` | clears `next_touch_due` (exits cadence); optionally sets status |
| **Giving up / nurture** | `dogfu crm touch stop <lead_id> [--status <Bad Fit or Nurture id>]` | same mechanism as reply; intent + status differ |
| Change **status only**, no cadence change | `dogfu crm lead update <lead_id> -s <status_id>` | funnel label only |

Notes on the verbs:
- `record` **auto-advances** from the lead's current stage — don't pass `--stage` unless you
  mean to force one (e.g. redoing a touch). Don't record the same touch twice.
- `record` defaults to **logging a note and creating a follow-up task** on the next-touch-due
  date — that's the BDR's automatic reminder, and what `touch history` reads back. Leave them
  on unless the BDR asks otherwise.
- `record` does **not** change status (status stays a human label). Use `--status` on
  `reply`/`stop`, or `lead update -s`, when the interaction also moves the funnel.
- `reply` vs `stop`: both clear the cadence and pull the lead from all queues. `reply` = they
  answered (→ Engaged); `stop` = we're done / nurturing (→ Bad Fit or Nurture). The
  difference is intent and which `--status` you set.

### Editing the records (leads, contacts, notes, tasks)

| Goal | Command |
| :-- | :-- |
| Create / update / delete a lead | `dogfu crm lead create -n "<name>" [-u <url>] [-d "<desc>"] [-s <status_id>]` · `lead update <lead_id> [-n] [-u] [-d] [-s]` · `lead delete <lead_id> [-y]` |
| Set curated company fields | on `lead create`/`update`: `--industry "<choice>"` (validated against Close's list) · `--employees <n>` · `--revenue <usd>` · `--business-model "<text>"` · `--seo-pages <n>` |
| Add / update / delete a contact (links go in `-u`, repeatable) | `dogfu crm contact create <lead_id> -n "<name>" [-t "<title>"] [-u <url>]... [-e <email>]... [-p <phone>]...` · `contact update <contact_id> [-n] [-t]` · `contact delete <contact_id> [-y]` |
| Add / list notes | `dogfu crm note create <lead_id> -t "<text>"` · `note list <lead_id> [-l N]` — note bodies are stored as plain text; don't rely on literal `<`/`>`/`&` for structure |
| Manage follow-up tasks | `dogfu crm task create -l <lead_id> -t "<text>" [-d YYYY-MM-DD] [-a <user_id>]` · `task list [-l <lead_id>] [-p]` · `task complete <task_id>` · `task update <task_id> [...]` · `task delete <task_id> [-y]` |

***

## Reporting the funnel / cadence load

There's no aggregate endpoint — compose it from list calls:

1. `crm status list` → the statuses and ids.
2. Per status: `crm lead list -s <id> -l <high> -o file` and count → leads in each funnel status.
3. Cadence load due *today*: `crm touch due --stage 0`, `--stage 1`, `--stage 2`, and a bare
   `crm touch due` (total) → how many are waiting for each touch right now.

Lead with the **number that needs work today**, then a funnel table (status → count) and a
cadence table (next touch → due-now count). **Counts are capped by `--limit`** — for a big
funnel, raise the limit or report "at least N" and say so.

***

## Operating rules

- **Resolve status ids first** with `crm status list`; map labels at runtime (fit/working →
  Qualified, replied → Engaged, dead → Bad Fit, later → Nurture). Never hardcode an id.
- **Translate phrasing to a stage carefully** — recall "second touch" = touch 2 = `--stage 1`,
  and that touch 0 (connection request) comes from the Qualified *status* list, not `touch
  due`. Ask one clarifying question when "first touch" is ambiguous.
- **Never assume a reply.** Only `touch reply` when the BDR states the lead responded.
- **Confirm before bulk mutations.** For any batch ("mark these five replied", "stop all of
  these"), echo the exact lead names/ids you're about to change and wait for a go-ahead, then
  do them **one call at a time**. Single, clearly-specified writes don't need a confirmation
  round-trip.
- **Don't double-record.** `record` advances from the current stage; running it twice skips a
  touch. Check `touch_stage` if unsure.
- **Put data where it belongs:** a person's LinkedIn/X in that contact's `-u` urls; reusable
  context in a note; reminders as tasks; the brief headline in the lead `description`.

## Response format

- **Work queue:** a numbered / tabular list — lead name + company, the contact(s) with their
  LinkedIn/X links, last touched, days overdue, and the **next action** (channel + which
  touch). Make it copy-paste actionable so the BDR can go message immediately.
- **After a write:** one line confirming what changed — lead, new stage/status, and the next
  due date (or "exited cadence").
- **Status/report:** the headline "needs work today" number first, then the funnel + cadence
  tables.
