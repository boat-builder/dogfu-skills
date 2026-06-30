# Opportunity spec — follow-up changes

The full `dogfu crm opportunity` build spec was handed to the CLI devs separately. These are the
**refinements decided afterward** — apply them on top of that spec; **where they differ, these
win.** Two audiences: **Close admin** (in the Close UI) and **CLI** (devs).

***

## Close admin

1. **Delete the `Trial` lead status.** With opportunities in place, "trial" is a deal *stage* (in
   the opportunity pipeline), not an account label — a lead stays **Engaged** while a Trial-stage
   opportunity runs. Clean delete: 0 leads were in it (checked 2026-06-30). If any land there
   first, move them to **Engaged** before deleting.

2. **Delete the seeded example opportunities.** They aren't used by anything (no opportunity
   support has shipped), so remove them before the real pipeline goes live to keep the
   pipeline/forecast clean. Opportunity deletes in Close are permanent — glance to confirm they're
   examples first.

Final lead statuses after this: Potential, Qualified, **Engaged**, Customer, Bad Fit, Not
Interested, Canceled, DNC.

***

## CLI (devs)

3. **`touch reply` defaults the lead status to Engaged** — so the skill never has to resolve or
   pass the Engaged id (this is the concrete design for the spec's "default `touch reply` →
   Engaged" item):
   - Add `cadence.REPLY_STATUS = "Engaged"` — a **label**, resolved to an id at runtime from the
     live status list (same pattern as the existing `REACH_OUT_STATUS`; never hardcode the id).
   - Give the exit helper a default: `exit_cadence(lead_id, status_id=None, default_status_label=None)`
     — when `status_id` is None **and** `default_status_label` is set, resolve label→id and apply
     it. If the label isn't configured in the account, end the cadence **without** a status move
     and append a line to the returned `TouchState.warnings` (don't error).
   - `touch reply` passes `default_status_label=cadence.REPLY_STATUS`; an explicit `--status <id>`
     still overrides. **`touch stop` gets no default** — the right terminal status (Bad Fit / Not
     Interested / Canceled) depends on *why* you're stopping, so it stays `--status`-only.
   - Net: `dogfu crm touch reply <lead_id>` ends the chase **and** sets Engaged in one call.
     Depends on the `Engaged` status existing in Close first.

4. **`cadence.TERMINAL_STATUSES`** — set it to
   `("Engaged", "Customer", "Bad Fit", "Not Interested", "Canceled", "DNC")`. **Correction to the
   original spec:** drop `Nurture` (not a live status) **and `Trial`** (the lead status is being
   deleted, item 1); add `Engaged`/`Canceled`/`DNC`. A lead in any of these has left the cold
   sequence, so an open `[dogfu:cadence]` task on it is an anomaly `reconcile` retires; `Engaged`
   keeps a handed-off lead out of the cold queue.

***

## Downstream (skills, once the CLI changes ship)

- Drop `--status <Engaged id>` from `lead-touch`'s reply row — the agent just runs `touch reply`.
- Fix `crm-cleanup`'s status classification: engaged = {Engaged} (no Trial); terminal — drop
  `Nurture`, add `Canceled`/`DNC`.
