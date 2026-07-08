# Intake — writing findings & deliverables into the CRM

The flow for persisting what another skill (or the user) produced: a researched lead with
its attributes, contacts, and write-up (from **lead-research**), or a deliverable URL such
as a published audit report (from **first-audit**). Load `records.md` alongside this — it
holds the commands, the curated attribute flags, and the placement rules; this file holds
the contract and the order of operations.

***

## Status comes from a human decision — never yours

A researched lead's status is set by the **user's pursue-or-drop answer**, made at the
researching skill's checkpoint. There are only two outcomes:

| Decision | Status | Note |
| :-- | :-- | :-- |
| pursue | **Qualified** | enters the outreach flow (the reach-out task opens on this status) |
| drop | **Bad Fit** | record the user's reason; competitors additionally get "COMPETITOR — do not contact" in the description **and** the note, so they're never re-prospected |

**No decision → no lead is written.** There is no auto-qualify and no parked bucket: if
the run couldn't get a pursue/drop answer (e.g. non-interactive), return the findings and
stop. Never write a status the user didn't choose. Resolve the status id at runtime
(`crm status list`).

***

## The upsert — one pass, everything found

The user's decision is the go-ahead for the whole write: run it in one pass and show what
you wrote — don't pause again per field. Upsert on the resolved domain:

1. **Dedup first:** `dogfu crm lead list -q "<domain>"` (or `-n "<company>"`) — if a match
   exists, `lead update` it; else `lead create`.
2. **The lead:** brief 1–2 line `-d` description, the decision's `-s <status_id>`, **plus
   every curated attribute flag you have a value for** (`records.md` lists them — never
   pass blanks; choices fields validate against Close's live options, a rejection echoes
   the allowed list — pick the closest or omit and note it).
3. **Contacts — every person found, whatever the decision:** `crm contact create <lead_id>
   -n "<name>" -t "<title>" -u <linkedin> -u <x> [-e <verified email>]` — links in the
   native `-u` urls, one call per person.
4. **The note — the depth:** `crm note create <lead_id> -t "<write-up>"` carrying what
   they do / who they serve, the market used, metrics **with sources** (labeled as
   estimates), the brief's read, the **user's decision + reason**, the date — and, for a
   pursued lead, the competitive ratios, DM hooks with evidence, and known gaps.

The researched company attributes map onto the curated flags by name (`records.md`
lists them); take the values from the findings already in the conversation — don't
re-research. This flow just requires that every known value lands in its flag, not in
prose.

**Never waste research** — a drop gets the same contacts, attributes, and note as a
pursue; only the status differs.

***

## Attaching a deliverable URL (published audit / report)

For recording a produced artifact (e.g. a first-audit report URL) on a lead:

1. **Find the lead.** Reuse a `lead_id` you already resolved this run; otherwise
   `dogfu crm lead list -q "<domain>"`. If none exists, create a minimal one —
   `crm lead create -n "<company>" -u <domain>` — **without a status** (attaching a
   deliverable is not a qualification decision).
2. **Attach as a note:** `dogfu crm note create <lead_id> -t "<deliverable>: <url> —
   <one-line summary>"`. A note keeps the history and is the default home; only also put
   it in the lead `description` if the user wants it on the headline.

***

## Failure modes

- `412` "no Close CRM API key configured" → Close isn't connected (Console → CRM
  Integration). Surface it and stop the CRM step — don't retry, and don't lose the
  findings: hand them back to the user (the deliverable/research still stands).
- A choices-flag rejection is not an error to retry blindly — the message lists the
  allowed values; pick the closest or omit and say so in the note.

After the write, confirm in one line what landed where (lead + status, N contacts, note,
URL) — that confirmation is the run's receipt.
