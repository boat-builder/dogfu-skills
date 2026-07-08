# Records — the shared read/edit surface

The base layer every CRM operation uses: finding and reading leads, and editing the
records that hang off them — contacts, notes, ad-hoc tasks, curated fields. Flow
mechanics live in their own references (`cold-outreach.md` for touch verbs, `deals.md`
for opportunity verbs, `cleanup.md` for the audit, `intake.md` for the research write).

All leaf commands take `-f json|table` (default json) and `-o FILE`.

## The Lead object

A `Lead` from any read carries:

- **Identity** — `name`, `url` (website), `description` (a short headline, 1–2 lines).
- **Status** — `status_id` + `status_label`. Typical labels: *Potential, Qualified,
  Connected, Engaged, Customer, Bad Fit, Not Interested, Canceled, DNC*. Ids are
  account-specific — resolve with `status list`, never hardcode.
- **Contacts** — embedded `contacts[]`; each person has `urls` (their LinkedIn / X —
  what you DM from), `emails`, `phones`, `title`. List calls can omit contacts — if a
  lead you're working has none embedded, `lead get <id>` to fetch them.
- **Outreach state** — system-managed cold-cadence fields: `touch_stage`,
  `last_touched`, `next_touch_due`, `touch_channel` (semantics in `cold-outreach.md`).
- **Curated company attributes** — the custom-field flags below; `null` when unset.

Opportunities, notes, and tasks are attached records with their own list/get commands.

## Reading state

| Goal | Command |
| :-- | :-- |
| List statuses + ids (do this first) | `dogfu crm status list` |
| Who am I (the Close user) | `dogfu crm whoami` |
| List / find leads (no filter = newest first; name / query / status narrow it — this is also how you search) | `dogfu crm lead list [-n "<name>"] [-q "<query>"] [-s <status_id>] [-l N] [--sort -date_created]` |
| Full lead incl. contacts + outreach fields | `dogfu crm lead get <lead_id>` |
| **The work queue** (open tasks due ≤ today, classified) | `dogfu crm worklist [--kind reach-out\|follow-up\|engage\|deal\|ad-hoc] [-l N]` |
| A lead's notes / tasks / contacts | `dogfu crm note list <lead_id> [-l N]` · `dogfu crm task list [-l <lead_id>] [-p]` · `dogfu crm contact list <lead_id>` · `contact get <contact_id>` |
| A choices field's allowed values | `dogfu crm field get "<name>"` (·`field list [-t lead\|contact\|opportunity]` for the schema) |

## Editing records

| Goal | Command |
| :-- | :-- |
| Create / update / delete a lead | `dogfu crm lead create -n "<name>" [-u <url>] [-d "<desc>"] [-s <status_id>] [curated flags]` · `lead update <lead_id> [-n] [-u] [-d] [-s] [curated flags]` · `lead delete <lead_id> [-y]` |
| Add / update / delete a contact (links go in `-u`, repeatable) | `dogfu crm contact create <lead_id> -n "<name>" [-t "<title>"] [-u <url>]... [-e <email>]... [-p <phone>]...` · `contact update <contact_id> [-n] [-t]` (name/title only — adding a URL means re-creating) · `contact delete <contact_id> [-y]` |
| Add a note | `dogfu crm note create <lead_id> -t "<text>"` — bodies are stored as plain text (HTML-escaped on write); don't rely on literal `<`/`>`/`&` for structure |
| Manage **ad-hoc** tasks (never a `[dogfu:*]`-tagged one) | `dogfu crm task create -l <lead_id> -t "<text>" [-d YYYY-MM-DD] [-a <user_id>]` · `task update <task_id> [...]` · `task complete <task_id>` · `task delete <task_id> [-y]` |
| Edit a choices field's options (**only on the user's explicit ask**) | `dogfu crm field add-choice <name> <choice>` · `field remove-choice <name> <choice> [-y]` |

## Curated company attributes (on `lead create` / `update`)

Set every flag you have a value for; skip the rest — never pass blanks. These are the
*only* custom fields dogfu sets.

- **Numbers:** `--employees` · `--marketing-team-size` · `--revenue` (USD) ·
  `--total-funding` (USD) · `--year-founded` · `--seo-pages` · `--organic-keywords`
- **Choices** (validated against Close's live options — a rejection echoes the allowed
  list; pick the closest or omit and note it): `--industry` · `--business-model` ·
  `--primary-market` · `--funding-stage` · `--seo-investment-tier` · `--seo-momentum` ·
  `--aeo-visibility`
- **Text:** `--company-linkedin` · `--company-x` (the company's own profile URLs)

## Where data belongs

- Lead `-u` = the company website; a **person's** LinkedIn/X go in that contact's `-u`
  urls (they render on the contact card).
- The lead `description` is a brief 1–2 line headline; **depth goes in a note** — the
  research write-up, what happened on a call, reusable context, audit trail.
- Structured company facts go in the curated attribute flags, not prose.
- Deal economics (value, deal type, confidence) live on the **opportunity**
  (`deals.md`) — never in the lead description.
- Reminders are **ad-hoc tasks**; what already happened is a note (or a touch, in the
  cold flow) — a task is future tense, a note/touch is past tense.
