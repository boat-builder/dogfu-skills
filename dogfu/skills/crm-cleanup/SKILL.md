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
  For pulling today's work queue (across all stages, read-only) use the lead-worklist skill;
  for recording outreach use lead-touch; for researching a new target, use lead-research.
---

# CRM cleanup — read-only health audit (powered by `dogfu`)

This skill is how a BDR (or an agent) **finds everything in the Close CRM that has drifted out
of the regular outreach flow**, before it silently rots. It is **read-only**: it composes
`dogfu crm` *read* calls, applies the model's invariants, and returns a report of anomalies —
each with the lead, the problem, and the exact command to fix it by hand. It never writes.

It is the audit counterpart to `lead-touch` (which *operates* the flow). The full model lives
in **lead-touch/SKILL.md**; this skill assumes it and only recaps what the checks need.

***

## The model, in one breath (see lead-touch for depth)

- A **lead is a company** with a **status** (funnel label: Potential, Qualified, Engaged,
  Customer, Bad Fit, Not Interested, Canceled, DNC) and **outreach state** in four system-managed
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

> **Note (reach-out tasks).** A reach-out is itself a real cadence task, opened when a lead
> enters the reach-out status. So a reach-out with **no** open cadence task **is** an anomaly —
> the CLI's `reconcile` flags it as `missing_reach_out_task`. The whole cadence-task invariant is
> owned by `reconcile` (Step 2A), not re-derived here.

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
dogfu crm touch reconcile -o reconcile.json            # cadence-task + deal-task anomalies (read-only)
dogfu crm status list -o statuses.json                 # {id ↔ label}; classify the funnel
# all leads, per status, with their cadence fields (raise -l for a big funnel):
dogfu crm lead list -s <status_id> -l 500 -o leads_<label>.json   # repeat per status
dogfu crm task list -p -o pending_tasks.json           # ALL open tasks (cadence + ad-hoc)
dogfu crm whoami -o me.json                            # your user id (for assignee checks)
```

`reconcile.json` (read-only without `--apply`) is the CLI's own audit of the **cadence-task and
deal-task invariant** — Step 2A just reports its rows. Everything else you compute from the lists.

From `statuses.json`, **classify labels at runtime** (account-specific — never hardcode ids):
*reach-out* = `Qualified`; *terminal* = {Bad Fit, Not Interested, Customer, Canceled, DNC};
*engaged* = {Engaged}; *potential* = {Potential}. Each `Lead` carries `status_label`,
`touch_stage`, `last_touched`, `next_touch_due`, `touch_channel`, and embedded `contacts[]`.

From `pending_tasks.json`, the **ad-hoc** tasks (text with no `[dogfu:cadence]` / `[dogfu:deal]`
tag) drive the ownership (E) and staleness checks. The tagged cadence/deal tasks are already
audited in `reconcile.json` (Step 2A) — don't re-group or re-check them by hand.

> **Scale & limits.** `lead list` caps at `--limit` (raise it, or report "≥ N" and say so).
> `task list` has **no limit flag** — it returns the provider's page of pending tasks; for a
> very large org, audit per-lead instead and note the coverage. Pull concurrently.

***

## Step 2 — Run the checks

Each check below lists **how to detect it**, **why it matters**, and the **fix** to suggest.
Severity: **🔴 blocker** (lead silently off the queue / being mis-worked) · **🟠 drift** (state
inconsistent, will mislead) · **🟡 hygiene** (quality/SLA). `[fix: CLI]` = a safe one-line
`dogfu` fix the BDR can run; `[fix: reconcile]` = a cadence-task/deal-task repair to leave for
`reconcile --apply` (don't hand-edit tagged tasks — the single-writer rule); `[fix: human]` =
needs a judgment call.

### A. Cadence-task & deal-task invariant — `reconcile` owns this

Don't re-derive these by hand: **`reconcile.json` already lists them.** The CLI computes the whole
invariant — every reach-out / in-cadence lead carries exactly one open cadence task, no exited or
terminal-status lead carries one, and ≤ one open `[dogfu:deal]` task per open opportunity (none on
a won/lost or vanished one). Report each row it returns; repair the bulk with **`dogfu crm touch
reconcile --apply`** (the single writer — never hand-complete a `[dogfu:cadence]` / `[dogfu:deal]`
task). The keys it can return:

| Key (from `reconcile`) | Means | Sev | Fix |
| :- | :- | :- | :- |
| `missing_reach_out_task`, `missing_cadence_task` | in the sequence but no open cadence task — due yet invisible in the task list | 🔴 | `[fix: reconcile]` |
| `cadence_task_after_exit`, `cadence_task_on_terminal_status` | reminder still open after the lead replied/stopped or went terminal | 🟠 | `[fix: reconcile]` (a terminal lead may also need its **status** set — see B) |
| `duplicate_cadence_tasks`, `duplicate_deal_tasks` | 2+ open reminders where exactly one is allowed | 🟠 | `[fix: reconcile]` |
| `cadence_task_due_drift` | the reminder's due date ≠ the lead's `next_touch_due` | 🟡 | `[fix: reconcile]` |
| `deal_task_orphaned`, `deal_task_on_closed_opportunity` | open deal task on a vanished or won/lost opportunity | 🟠 | `[fix: reconcile]` |

### B. Status ↔ sequence drift (the lens `reconcile` doesn't have)

| # | Anomaly | Detect | Sev | Fix |
| :- | :- | :- | :- | :- |
| B1 | **Exited but funnel not moved** — replied/stopped yet still Qualified/Potential | lead `touch_stage` not null **and** `next_touch_due` empty **and** `status_label` ∈ {Qualified, Potential} | 🟠 | `[fix: human]` set the real status: `dogfu crm lead update <lead_id> -s <Engaged\|Bad Fit\|Not Interested id>` |
| B2 | **Engaged/Customer still in cadence** — won/replied but still scheduled | `status_label` ∈ {Engaged, Customer} **and** `next_touch_due` set (a still-set due date reads as "in cadence", so `reconcile` won't flag the task — this status lens catches it) | 🟠 | `[fix: CLI]` `dogfu crm touch stop <lead_id>` (or `touch reply`) to clear the sequence |

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
| D2 | **Chase fatigue** — many touches, no reply | `touch_stage` ≥ **5** **and** still in cadence (`next_touch_due` set) | 🟡 | `[fix: human]` decide: keep nudging or `dogfu crm touch stop <lead_id> --status <Not Interested id>` |

### E. Ownership

| # | Anomaly | Detect | Sev | Fix |
| :- | :- | :- | :- | :- |
| E1 | **Unassigned task** | open task with empty `assigned_to` | 🟡 | `[fix: CLI]` `dogfu crm task update <task_id> -a <user_id>` (your id from `whoami`) |

***

## Scope & limits — be honest about these

- **Read-only.** This skill never writes. It reports anomalies and the command to fix each.
  **Every cadence-task / deal-task repair (Section A)** is applied by `dogfu crm touch reconcile
  --apply` — don't hand-create or hand-complete a `[dogfu:cadence]` / `[dogfu:deal]` task (the
  single-writer rule keeps outreach state sane). B2, C3, E1 are safe one-line `dogfu` fixes; the
  rest need a human judgment call.
- **Not yet detectable (data the documented CLI surface doesn't expose):**
  - *Age-based staleness* of reach-outs or Potential leads (created/qualified date) — the
    canonical `Lead` has no created date. Approximate via `dogfu crm lead list -s <id> --sort
    date_created` (oldest first) and eyeball the head. D1/D2 work because they key off
    `next_touch_due` / `touch_stage`, which **are** exposed.
  - *Task assigned to a deactivated/unknown user* — there's no user-list command (only
    `whoami`), so you can flag empty assignees (E1) but not invalid ones.
  - *Lead owner missing* — not exposed on the model; use task assignee (E1) as the proxy.
- **Now covered (shipped):** reach-out tasks and the deal-task invariant are both audited by
  `reconcile` (Section A) — flag them. The richer **opportunity-health** checks (an Engaged lead
  with no opportunity, a stalled or under-specified deal) belong to `lead-engage` /
  `crm opportunity due`, not this read-only audit.

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
- **Let `reconcile` own the tagged-task invariant.** Don't re-derive the cadence/deal-task checks
  by hand — read them from `reconcile` (Step 2A). Identify ad-hoc tasks (for E) by the *absence*
  of a `[dogfu:cadence]` / `[dogfu:deal]` tag; never treat one as the other.
- **Gather broad, drill narrow.** Pull list endpoints once to files; only `lead get` /
  `note list` / `contact list` the leads a cheap check already flagged.
- **Respect the single-writer rule.** Route every cadence/deal-task repair to
  `reconcile --apply`. Only suggest hand-fixes that are safe (B2, C3, E1) or clearly a human decision.
- **Never write.** If the user wants the fixes applied, hand off to `lead-touch` (per-lead) or
  run `reconcile --apply` (bulk cadence/deal tasks) — say which.
