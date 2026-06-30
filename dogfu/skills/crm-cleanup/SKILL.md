---
name: crm-cleanup
description: >-
  Audit Agent Berlin's Close CRM for leads and tasks that have fallen out of the regular
  outreach flow — using the `dogfu` CLI, read-only. Use this whenever someone wants a CRM
  health check or cleanup pass: "audit the CRM", "what's broken / inconsistent in Close",
  "find leads that fell out of the flow", "find orphaned / duplicate cadence tasks", "which
  leads are stuck", "find leads being chased that already replied or are Bad Fit", "find
  qualified leads with no contacts / no research / no website", "find duplicate leads",
  "overdue follow-ups", "unassigned tasks", or "reconcile the outreach state". It detects and
  reports anomalies (and the exact command to fix each); it does **not** write to the CRM.
  For pulling today's work queue or recording outreach, use the lead-touch skill; for
  researching a new target, use lead-research.
---

# CRM cleanup — read-only health audit (powered by `dogfu`)

This skill is how a BDR (or an agent) **finds everything in the Close CRM that has drifted out
of the regular outreach flow**, before it silently rots. It is **read-only**: it composes
`dogfu crm` *read* calls, applies the model's invariants, and returns a report of anomalies —
each with the lead, the problem, and the exact command to fix it by hand. It never writes.

It is the audit counterpart to `lead-touch` (which *operates* the flow). The full model lives
in **lead-touch/SKILL.md**; this skill assumes it and only recaps what the checks need.

> **Why this exists.** BDRs aren't supposed to edit Close directly, but humans make mistakes
> and some state is computed, not stored — so leads quietly fall off the queue: a reminder
> closed by hand, a lead still chased after it replied, a Qualified lead with nobody to
> message. This skill surfaces those. A future `dogfu crm touch reconcile --fix` will repair
> them automatically; until then, this finds them and tells you the fix.

***

## The model, in one breath (see lead-touch for depth)

- A **lead is a company** with a **status** (funnel label: Potential, Qualified, Engaged/Trial,
  Customer, Bad Fit, Not Interested, Nurture) and **outreach state** in four system-managed
  lead fields: `touch_stage` (last completed touch, `null` = none), `last_touched`,
  `next_touch_due` (empty = **out of the sequence**), `touch_channel` (channels tried).
- **The cadence task** is the one CLI-owned "next action" reminder. Its task `text` carries the
  tag **`[dogfu:cadence]`** — that substring is how you tell it from an ad-hoc task. The rule
  is **exactly one open cadence task per lead while it's in the sequence**.
- **In the sequence** = `next_touch_due` is set (a follow-up is scheduled) *or* the lead is a
  reach-out (Qualified + `touch_stage` null). **Out** = replied/stopped (`next_touch_due`
  empty after a touch).
- **The work queue** (`dogfu crm touch due`) = reach-outs + follow-ups due. The audit checks
  the integrity of everything *behind* that queue.

> **Note (reach-out tasks).** Today a reach-out has **no** task object — it's computed from
> status. So "reach-out with no cadence task" is **normal, not an anomaly**, and this skill
> does **not** flag it. Once the CLI creates reach-out tasks on qualification (the planned R1
> change), that check turns on. Everything else below works now.

***

## Running `dogfu`

Same dependency and auth as the other skills. `dogfu` is a published CLI (`pip install dogfu`)
authenticated through the **dogfu MCP**; output is **JSON by default** (read stdout; `-o FILE`
for big lists). If a `crm` call returns an auth error (`missing dogfu token`, `invalid or
expired token`), call the MCP's **`get_setup_instructions`** and follow it
(`dogfu configure --otp <OTP> --title "CRM cleanup"`). A `412` "no Close CRM API key
configured" means the Close key isn't set in the admin **Console → CRM Integration** — surface
that rather than retrying. Read calls are independent HTTP requests — **run them concurrently**.

***

## Step 1 — Gather state once, then compute in memory

Pull broad lists **once**, to files, and do the cross-referencing yourself. Only drill into a
single lead (`lead get` / `note list` / `contact list`) for leads a cheap check already
flagged — never per-lead up front.

```bash
dogfu crm status list -o statuses.json                 # {id ↔ label}; classify the funnel
# all leads, per status, with their cadence fields (raise -l for a big funnel):
dogfu crm lead list -s <status_id> -l 500 -o leads_<label>.json   # repeat per status
dogfu crm task list -p -o pending_tasks.json           # ALL open tasks (cadence + ad-hoc)
dogfu crm whoami -o me.json                            # your user id (for assignee checks)
```

From `statuses.json`, **classify labels at runtime** (account-specific — never hardcode ids):
*reach-out* = `Qualified`; *terminal* = {Bad Fit, Not Interested, Customer, Nurture}; *engaged*
= {Engaged, Trial}; *potential* = {Potential}. Each `Lead` carries `status_label`,
`touch_stage`, `last_touched`, `next_touch_due`, `touch_channel`, and embedded `contacts[]`.

From `pending_tasks.json`, **split open tasks** into **cadence** (text contains
`[dogfu:cadence]`) and **ad-hoc** (everything else), and **group by `lead_id`**. That grouping
drives every cadence-task check below.

> **Scale & limits.** `lead list` caps at `--limit` (raise it, or report "≥ N" and say so).
> `task list` has **no limit flag** — it returns the provider's page of pending tasks; for a
> very large org, audit per-lead instead and note the coverage. Pull concurrently.

***

## Step 2 — Run the checks

Each check below lists **how to detect it**, **why it matters**, and the **fix** to suggest.
Severity: **🔴 blocker** (lead silently off the queue / being mis-worked) · **🟠 drift** (state
inconsistent, will mislead) · **🟡 hygiene** (quality/SLA). `[fix: CLI]` = a safe one-line
`dogfu` fix the BDR can run; `[fix: reconcile]` = a cadence-task repair to leave for
`reconcile --fix` (don't hand-edit cadence tasks — the single-writer rule); `[fix: human]` =
needs a judgment call.

### A. Cadence-task invariant (the core)

| # | Anomaly | Detect | Sev | Fix |
| :- | :- | :- | :- | :- |
| A1 | **Orphaned reminder** — in the sequence but no open cadence task | lead `next_touch_due` set **and** no open `[dogfu:cadence]` task for its `lead_id` | 🔴 | `[fix: reconcile]` — the lead is due but invisible in the task list |
| A2 | **Duplicate reminders** — 2+ open cadence tasks | `lead_id` with ≥2 open `[dogfu:cadence]` tasks | 🟠 | `[fix: reconcile]` (collapses to one; or `touch record` self-heals on next touch) |
| A3 | **Lingering reminder** — task open after the lead exited | open `[dogfu:cadence]` task whose lead has `next_touch_due` **empty** | 🟠 | `[fix: CLI]` `dogfu crm task complete <task_id>` (lead already exited — safe) |
| A4 | **Chasing a dead lead** — cadence task on a terminal status | open `[dogfu:cadence]` task whose lead `status_label` ∈ terminal | 🔴 | `[fix: CLI]` `dogfu crm touch stop <lead_id>` (ends chase, closes task) |
| A5 | **Due-date drift** — reminder date ≠ outreach state | open cadence task `due_date` ≠ lead `next_touch_due` | 🟡 | `[fix: CLI]` `dogfu crm task update <task_id> -d <next_touch_due>` |

### B. Status ↔ sequence drift

| # | Anomaly | Detect | Sev | Fix |
| :- | :- | :- | :- | :- |
| B1 | **Exited but funnel not moved** — replied/stopped yet still Qualified/Potential | lead `touch_stage` not null **and** `next_touch_due` empty **and** `status_label` ∈ {Qualified, Potential} | 🟠 | `[fix: human]` set the real status: `dogfu crm lead update <lead_id> -s <Engaged\|Nurture\|Bad Fit id>` |
| B2 | **Engaged/Customer still in cadence** — won/replied but still scheduled | `status_label` ∈ {Engaged, Trial, Customer} **and** (`next_touch_due` set **or** open cadence task) | 🟠 | `[fix: CLI]` `dogfu crm touch stop <lead_id>` (or `touch reply`) to clear the sequence |

### C. Outreach-readiness gaps (queue says act, but you can't)

| # | Anomaly | Detect | Sev | Fix |
| :- | :- | :- | :- | :- |
| C1 | **No one to message** — Qualified, zero contacts | Qualified lead with empty `contacts[]` (confirm with `dogfu crm lead get <id>` — list calls can omit contacts) | 🔴 | `[fix: human]` add a contact (re-run `lead-research`, or `crm contact create`) |
| C2 | **No channel** — a contact with no link/email/phone | a contact on a Qualified lead with empty `urls` **and** `emails` **and** `phones` | 🟠 | `[fix: human]` enrich the person (note: `contact update` only edits name/title — adding a URL means re-creating the contact) |
| C3 | **No website** — broken dedup/research key | lead `url` empty | 🟡 | `[fix: CLI]` `dogfu crm lead update <lead_id> -u <domain>` |
| C4 | **Duplicate leads** — same company twice | group all leads by normalized domain (strip scheme/`www`); count > 1 | 🟠 | `[fix: human]` merge in the Close UI (no CLI merge); keep the richer record |
| C5 | **Qualified, no research note** | Qualified lead whose `dogfu crm note list <id>` is empty | 🟡 | `[fix: human]` run `lead-research`, or add the write-up |

### D. Staleness / neglect SLAs (thresholds configurable; defaults below)

| # | Anomaly | Detect | Sev | Fix |
| :- | :- | :- | :- | :- |
| D1 | **Badly overdue follow-up** | lead `next_touch_due` ≤ today − **14d** | 🟠 | `[fix: human]` act now (it's in `touch due`), or `touch stop` if abandoning |
| D2 | **Chase fatigue** — many touches, no reply | `touch_stage` ≥ **5** **and** still in cadence (`next_touch_due` set) | 🟡 | `[fix: human]` decide: keep nudging or `dogfu crm touch stop <lead_id> --status <Nurture id>` |

### E. Ownership

| # | Anomaly | Detect | Sev | Fix |
| :- | :- | :- | :- | :- |
| E1 | **Unassigned task** | open task with empty `assigned_to` | 🟡 | `[fix: CLI]` `dogfu crm task update <task_id> -a <user_id>` (your id from `whoami`) |

***

## Scope & limits — be honest about these

- **Read-only.** This skill never writes. It reports anomalies and the command to fix each.
  **Cadence-task repairs (A1/A2)** should wait for `dogfu crm touch reconcile --fix` — don't
  hand-create or hand-delete cadence tasks (the single-writer rule keeps outreach state sane).
  A3–A5, B2, C3, E1 are safe one-line fixes; the rest need a human judgment call.
- **Not yet detectable (data the documented CLI surface doesn't expose):**
  - *Age-based staleness* of reach-outs or Potential leads (created/qualified date) — the
    canonical `Lead` has no created date. Approximate via `dogfu crm lead list -s <id> --sort
    date_created` (oldest first) and eyeball the head, or leave it to `reconcile`. D1/D2 work
    because they key off `next_touch_due` / `touch_stage`, which **are** exposed.
  - *Task assigned to a deactivated/unknown user* — there's no user-list command (only
    `whoami`), so you can flag empty assignees (E1) but not invalid ones.
  - *Lead owner missing* — not exposed on the model; use task assignee (E1) as the proxy.
- **Deferred checks:** reach-out-with-no-task is normal pre-R1 (don't flag). Opportunity checks
  (Engaged lead with no opportunity, stalled/under-specified opportunity) wait for the
  Engaged/opportunity phase.

***

## Response format

Lead with the **headline**: total anomalies and how many are 🔴 blockers ("3 leads are silently
off the queue"). Then a section per severity (blockers first), each finding as a row:

> **<Lead name>** (`<lead_id>`) — <one-line problem> · **Fix:** `<command>` (or "await
> reconcile" / "human decision: …")

Group identical anomalies so the BDR can batch them. Make every fix copy-paste runnable. If a
list was capped by `--limit`, say so and give the real floor ("≥ N"). Close with a one-line
**summary table** (anomaly → count) so repeated runs show whether the CRM is getting cleaner.

## Operating rules

- **Resolve status ids at runtime** (`crm status list`); classify labels (reach-out / terminal
  / engaged / potential) — never hardcode ids.
- **Identify cadence tasks only by the `[dogfu:cadence]` tag**; never treat an ad-hoc task as
  one (or vice-versa).
- **Gather broad, drill narrow.** Pull list endpoints once to files; only `lead get` /
  `note list` / `contact list` the leads a cheap check already flagged.
- **Respect the single-writer rule.** Detect cadence-task problems; route their repair to
  `reconcile --fix`. Only suggest hand-fixes that are safe (A3–A5, B2, C3, E1) or clearly a
  human decision.
- **Never write.** If the user wants the fixes applied, hand off to `lead-touch` (per-lead) or
  wait for `reconcile --fix` (bulk) — say which.
