# Opportunity support — Close setup + `dogfu crm opportunity` CLI spec

`lead-engage` drives **Close Opportunities**, which the `dogfu` CLI does not support yet
(verified against the CLI source at `backend/dogfu`, v0.10.0 — the `crm` group has lead /
contact / task / note / status / touch / whoami, but no `opportunity`). This file is the
implementation target for three pieces of work:

1. **Close configuration** — a lead status, an opportunity pipeline, an opportunity custom
   field (done once, in the Close admin UI / API).
2. **Backend** — allowlist the opportunity + pipeline endpoints on the admin CRM proxy.
3. **CLI** — the `dogfu crm opportunity …` surface (built in the `dogfu` CLI repo —
   *not* this skills repo, which only ships the skill markdown).

It is written to match the CLI's existing patterns so the implementer can follow them, and so
the skill author can keep `SKILL.md`'s command surface in sync.

> **Architecture reminder.** The CLI does **not** talk to Close directly. It calls the dogfu
> backend admin proxy (`config.DEFAULT_BASE_URL` → `/api/v1/admin/crm/*`) which holds the Close
> key and forwards. So every new opportunity call needs a **backend route allowlisted** before
> the CLI can reach it — opportunity support is a backend + CLI change, not CLI-only.

***

## 1. Close configuration (do this first)

### 1a. Add the `Engaged` lead status

Insert a lead status **`Engaged`** between **Qualified** and **Customer**. The funnel becomes:

```
Potential → Qualified → Engaged → Customer        (+ Bad Fit / Not Interested / Canceled / DNC)
```

`Engaged` = "replied / in active conversation, deal not yet qualified-and-open" — the home for a
lead between the cold reply and an opportunity. Once it exists, `touch reply` defaults the lead
to it (see §6).

> **Remove the `Trial` lead status.** With opportunities in place, "trial" is a deal *stage*,
> not an account label — the lead stays **Engaged** while a Trial-stage opportunity runs. Delete
> the `Trial` **lead** status. It's a clean delete (0 leads were in it as of 2026-06-30); if any
> land there first, move them to **Engaged** before deleting. (This also keeps it out of the
> cadence config in §6.)

### 1b. Create the opportunity pipeline

Create one pipeline (e.g. **`Sales`**) with these statuses, in order, with the right Close
**status type** (`active` / `won` / `lost`):

| Stage label | Close status type |
| :-- | :-- |
| Discovery | active |
| Trial | active |
| Proposal | active |
| Won | won |
| Lost | lost |

Close opportunity statuses are pipeline-scoped and have ids (`stat_…`) — resolve them at
runtime, never hardcode (mirrors `crm status list` for lead statuses).

### 1c. `Deal Type` opportunity custom field

Add an opportunity **custom field** `Deal Type` with choices **`Co-Pilot`** / **`Fully-Run`**
(Berlin's two engagement shapes). Close opportunities are first-class objects with their own
custom fields, so this is the right home (unlike Close *tasks*, which support no custom
fields/types/tags). Written via `custom.<field_id>` on the opportunity API.

***

## 2. Canonical models (CLI return shapes)

Follow `models/crm.py`: dataclasses subclassing `Model`, snake_case, Close quirks hidden behind
stable names. Add `Opportunity` and `Pipeline` (+ `PipelineStatus`):

```python
@dataclass
class PipelineStatus(Model):
    id: Optional[str] = None
    label: Optional[str] = None
    type: Optional[str] = None          # "active" | "won" | "lost"
    order: Optional[int] = None

@dataclass
class Pipeline(Model):
    id: Optional[str] = None
    name: Optional[str] = None
    statuses: list[PipelineStatus] = field(default_factory=list)

@dataclass
class Opportunity(Model):
    id: Optional[str] = None
    lead_id: Optional[str] = None
    lead_name: Optional[str] = None
    pipeline_id: Optional[str] = None
    pipeline_name: Optional[str] = None
    status_id: Optional[str] = None
    status_label: Optional[str] = None      # current stage, e.g. "Trial"
    status_type: Optional[str] = None       # active | won | lost
    value: Optional[float] = None           # MAJOR units (the CLI converts Close cents)
    value_period: Optional[str] = None      # one_time | monthly | annual
    value_currency: Optional[str] = None
    value_formatted: Optional[str] = None   # e.g. "$1,500 / month"
    deal_type: Optional[str] = None         # from the Deal Type custom field
    confidence: Optional[int] = None        # 0–100
    expected_close: Optional[str] = None     # ISO date (Close `date_close`)
    date_won: Optional[str] = None
    note: Optional[str] = None
    date_created: Optional[str] = None
    date_updated: Optional[str] = None
    # The single open next-step task on this opp (the [dogfu:deal] task, §5), if any.
    next_step: Optional[Task] = None        # reuse the existing Task model
    # Set by mutating verbs that self-heal a double-open deal task (mirrors TouchState.warnings).
    warnings: Optional[list[str]] = None
```

Normalize Close → canonical in `normalize/close.py` (e.g. `opportunity()`, `pipeline()`), the
same layer that already hides `display_name` / `note_html` / task `date`.

***

## 3. Command surface (`commands/crm.py`)

A new `@crm.group() opportunity` with leaf commands; all take `-f json|table` (default json),
`-o FILE`, and the hidden `--raw`. Confirm-before-write applies to every mutating verb.

### Reading
- **`pipelines`** → `Pipeline[]`. Resolve stage ids here. *(Read-only.)*
- **`list [-l <lead_id>] [--open] [--won] [--lost] [--stage <name|id>] [--pipeline <name|id>] [--limit N]`** → `Opportunity[]`. `--open` = status_type active.
- **`get <opp_id>`** → one `Opportunity`.
- **`due [--limit N]`** → the **deal queue** (the opportunity analogue of `touch due`): open
  opps whose open `[dogfu:deal]` task is due ≤ today, **plus** open opps with **no** open
  `[dogfu:deal]` task (dropped balls), most-urgent first, each tagged `due` / `no-next-step`;
  include `stalled` (no `date_updated` change in > 14 days) if feasible.

### Writing — open, advance, close, next-step
- **`create <lead_id> [--pipeline <name|id>] [--stage <name|id>] --value <n> [--period one_time|monthly|annual] [--currency USD] [--deal-type co-pilot|fully-run] [--confidence 0-100] [--close YYYY-MM-DD] [--note "<text>"] [--next "<action>" --next-due YYYY-MM-DD]`**
  → creates the opportunity. Defaults: first/only pipeline; stage = first `active` status
  (**Discovery**); period `monthly`. With `--next`, opens the first `[dogfu:deal]` task. *Does
  not touch the lead status.*
- **`advance <opp_id> --stage <name|id> [--next "<action>" --next-due YYYY-MM-DD]`** → set the opp
  to another `active` stage; optionally swap the next-step task in the same call. Alias for
  `update --stage`.
- **`next <opp_id> -t "<action>" -d <YYYY-MM-DD>`** → **the single-writer next-step verb**: close
  the opp's open `[dogfu:deal]` task and open a new one (§5). The skill calls this, never raw `task`.
- **`update <opp_id> [--value] [--period] [--currency] [--deal-type] [--confidence] [--close] [--note] [--stage]`** → patch economics/metadata.
- **`win <opp_id> [--value <final>] [--close YYYY-MM-DD]`** → set status to the pipeline's `won`
  status; stamp `date_won`; **close the open `[dogfu:deal]` task** (deal exited). Does **not**
  change the lead status — the skill sets **Customer** explicitly (visible/confirmable).
- **`lose <opp_id> [--reason "<text>"]`** → set status to the `lost` status; close the open
  `[dogfu:deal]` task; store reason. Skill sets the terminal lead status separately.
- **`delete <opp_id> [-y]`** → delete (rare).

> **Why `win`/`lose` don't auto-set the lead status:** the lead-status move (→ Customer / Not
> Interested / Canceled) is a separate, human-meaningful decision the skill shows for
> confirmation. Auto-coupling would hide a status change behind a deal verb. A `--set-lead-status`
> convenience can be added later, opt-in.

### SDK (`sdk/crm.py`)
Flat methods in lock-step with the CLI: `list_pipelines`, `list_opportunities`,
`get_opportunity`, `opportunities_due`, `create_opportunity`, `advance_opportunity`,
`set_next_step`, `update_opportunity`, `win_opportunity`, `lose_opportunity`,
`delete_opportunity`. Mutations return the affected `Opportunity`; `delete` returns None.

***

## 4. Close API / proxy mapping (for the implementer)

- **Opportunities:** `POST/GET/PUT /api/v1/opportunity/` (filter `?lead_id=…`); fields
  `lead_id`, `status_id`, `pipeline_id`, `value` (**cents** — convert to/from the model's major
  units at the boundary), `value_period`, `confidence`, `date_close`, `date_won`, `note`, and
  `custom.<deal_type_field_id>`. Reached via the admin proxy, so add `/api/v1/admin/crm/opportunity*`.
- **Pipelines & statuses:** `GET /api/v1/pipeline/` → pipelines with `statuses[]` (`id`, `label`,
  `type`). Add `/api/v1/admin/crm/pipeline*` to the proxy allowlist.
- **Win/Lost:** no separate endpoint — set `status_id` to a status whose `type` is `won`/`lost`;
  Close stamps `date_won` on won.
- **Deal-type custom field:** extend the curated-field plumbing (`crm_fields.py` `FieldSpec` +
  `CrmClient.resolve_lead_fields`) with an opportunity equivalent (`OPP_FIELDS` / `resolve_opportunity_fields`),
  validating `--deal-type` against the live field's choices exactly like the lead `--industry` flag.

***

## 5. The `[dogfu:deal]` next-step task (single-writer — model on `cadence.py`)

The deal's next step is the warm-phase analogue of the cold cadence task, and must follow the
**same single-writer discipline** that `cadence.py` already implements for `[dogfu:cadence]`.
Because Close tasks support **no tags/custom fields** (only text/date/is_complete/assigned_to),
identity is a **text tag**, exactly like the cadence one:

- Tag constant (new, alongside `cadence.CADENCE_TASK_TAG`): **`DEAL_TASK_TAG = "[dogfu:deal]"`**.
- **One open `[dogfu:deal]` task per open opportunity.** `opportunity next` (and `--next` on
  `create`/`advance`) is the single writer: it closes the prior open deal task and opens the new
  one — reuse the cadence swap/self-heal helper (`_close_open_cadence_task` → a shared
  `_close_open_tagged_task(tag)`). `win`/`lose` close it.
- Unlike the cadence task, **content and due date are rep-authored** (a deal's next step isn't a
  computed wait), so there's no wait-curve — just the swap discipline.
- `opportunity due` and `Opportunity.next_step` key off this tag.

> **R7 seam.** This is exactly the "route all task lifecycle through one owner" point from the
> cadence-task reliability work: cadence owns `[dogfu:cadence]`, opportunities own `[dogfu:deal]`,
> and both go through one tagged-task swap/dedup helper so the invariant ("≤ one open CLI task of
> each kind per record") holds and `reconcile` can audit both.

***

## 6. Config / status changes (`config.py` + `cadence.py`)

After step 1, the live lead statuses are Potential, Qualified, **Engaged**, Customer, Bad Fit,
Not Interested, **Canceled**, **DNC** (Trial removed). These have drifted from `cadence.py`'s
assumptions — fix when shipping:

- `cadence.REACH_OUT_STATUS` = `"Qualified"` — unchanged.
- `cadence.TERMINAL_STATUSES` is currently `("Customer", "Bad Fit", "Not Interested", "Nurture")`.
  Set it to **`("Engaged", "Customer", "Bad Fit", "Not Interested", "Canceled", "DNC")`** — drop
  `Nurture` (not a live status), add `Engaged`/`Canceled`/`DNC` (no `Trial`, since the lead status
  is gone). Rationale: a lead in any of these has left the *cold* sequence, so an open
  `[dogfu:cadence]` task on it is an anomaly `reconcile` retires. Adding `Engaged` is what keeps a
  handed-off lead out of the cold queue.

### `touch reply` defaults the status to Engaged (the "nicety")

So the skill never has to resolve or pass the Engaged id, make `touch reply` move the lead to
**Engaged** by default:
- Add `cadence.REPLY_STATUS = "Engaged"` — a **label**, resolved to an id at runtime from the
  live status list (same pattern as `REACH_OUT_STATUS`; never hardcode the id).
- Give the exit helper a default: `exit_cadence(lead_id, status_id=None, default_status_label=None)`
  — when `status_id` is None and `default_status_label` is set, resolve that label → id and apply
  it; if the label isn't configured in the account, end the cadence **without** a status move and
  append a line to the returned `TouchState.warnings` (don't error).
- `touch reply` passes `default_status_label=cadence.REPLY_STATUS`; an explicit `--status <id>`
  still overrides. **`touch stop` gets no default** — the right terminal status (Bad Fit / Not
  Interested / Canceled) depends on *why* you're stopping, so it stays `--status`-only.

Result: `dogfu crm touch reply <lead_id>` ends the chase **and** sets Engaged in one call, with no
id handling in the skill. Once it ships, the `lead-touch` skill drops `--status <Engaged id>` from
its reply row — the agent just runs `touch reply`.

***

## 7. Build checklist

- [ ] **Close:** add the `Engaged` lead status; **delete the `Trial` lead status**; create the `Sales` opportunity pipeline (Discovery → Trial → Proposal → Won/Lost); add the `Deal Type` opportunity custom field.
- [ ] **Backend:** allowlist `/api/v1/admin/crm/opportunity*` and `/api/v1/admin/crm/pipeline*`.
- [ ] **CLI models/normalize:** `Opportunity`, `Pipeline`, `PipelineStatus` + Close→canonical normalizers; value cents↔major.
- [ ] **CLI sdk/commands:** the `opportunity` group (§3) + flat SDK methods; runtime pipeline/status id resolution; `--deal-type` validated via opportunity custom-field plumbing.
- [ ] **CLI tasks:** `DEAL_TASK_TAG` + shared single-writer swap helper; `opportunity next`; close-on-win/lose; `due` + `Opportunity.next_step` keyed off the tag.
- [ ] **CLI config:** set `cadence.TERMINAL_STATUSES = ("Engaged","Customer","Bad Fit","Not Interested","Canceled","DNC")` (drop `Nurture`/`Trial`).
- [ ] **CLI reply default:** add `cadence.REPLY_STATUS = "Engaged"` + the `exit_cadence` default-status logic so `touch reply` sets Engaged with no `--status` (`touch stop` stays manual).
- [ ] **reconcile:** extend the audit to the `[dogfu:deal]` invariant (≤ one open per open opp).
- [ ] **Skills (once shipped):** drop `--status <Engaged id>` from `lead-touch`'s reply row; fix `crm-cleanup`'s status classification (engaged = {Engaged}; terminal: drop `Nurture`, add `Canceled`/`DNC`); drop the lead-engage README's dependency note.
