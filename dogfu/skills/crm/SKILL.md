---
name: crm
description: >-
  The single entry point for Agent Berlin's Close CRM, operated through the `dogfu` CLI.
  Use for ANY read or write to or from the CRM: look up a lead or contact ("what do we
  know about X"), pull today's worklist ("what should I work on"), work the cold
  outreach cadence (reach-outs / follow-ups due, record a touch the BDR sent, mark
  replied / not interested, touch history, funnel report), work live deals (land the
  discovery call, open / advance / close an opportunity, set next steps, inbound intake,
  pipeline / forecast), audit or clean up the CRM ("what's broken in Close", stuck
  leads, reconcile), save research findings or a published report URL onto a lead
  (e.g. right after a lead-research or first-audit run — "update the CRM with this"),
  or draft an email / DM / message for a lead from what the CRM knows. This file is
  deliberately tiny: it holds the shared model and routes each operation to one or two
  reference files — load only what the ask needs.
---

# CRM — Agent Berlin's Close CRM, via `dogfu`

Everything to and from the CRM goes through this skill. The `dogfu` CLI holds the Close
credentials and commands; this file holds the shared model and the routing table. Read
the reference file(s) for the operation in play — nothing else.

## Run `dogfu`

`dogfu` is a published CLI (`pip install dogfu`). **Authenticate first:** call the dogfu
MCP's `get_setup_instructions` tool and follow it — install the CLI if needed, then
`dogfu configure --otp <OTP> --title "CRM: <who/what>"` with the OTP it returns.

- Commands: `dogfu crm <command> [flags]`. Output is JSON by default — read stdout; add
  `-o FILE` for large lists and read back only what you need. `-f table` is for humans.
- A `412` / "no Close CRM API key configured" means the admin hasn't connected Close
  (Console → CRM Integration) — surface it, don't retry.
- Read calls are independent HTTP requests — run them concurrently when it helps.

## The model in one breath

- A **lead is a company** (not a person). It carries identity (`name`, `url`,
  `description`), a **status** (funnel label), **contacts** (the people, with their
  LinkedIn/X in `urls`), system-managed **outreach state** (cold-cadence fields), zero
  or more **opportunities** (the deals you forecast), curated **company attributes**,
  and attached **notes** + **tasks**.
- **Three layers — never conflate:** *status* = funnel position (a human label);
  *outreach state* = cold-sequence position (machine-tracked; touches never change
  status); *opportunity* = the deal itself.
- The lifecycle: `Qualified —cold cadence: reach-out → follow-ups—▶ reply → Connected
  —discovery call confirms a deal—▶ Engaged (opportunity: Discovery → Trial → Proposal)
  —won—▶ Customer` (exits anywhere → Bad Fit / Not Interested / Canceled / DNC).
- **Tasks tagged `[dogfu:cadence]` / `[dogfu:engage]` / `[dogfu:deal:<opp_id>]` are
  CLI-owned — never hand-create or hand-complete one.** The CLI is their single writer
  (`touch` / `opportunity` verbs and status moves); `dogfu crm reconcile` audits and
  repairs them. You create only untagged **ad-hoc** tasks, on request.
- **`dogfu crm worklist`** is the whole work queue — every open task due ≤ today,
  classified `reach-out | follow-up | engage | deal | ad-hoc` with a human
  `next_action`. It answers "what should I work on" across all phases; `--kind <k>`
  slices it.

## Route the ask — read only what it needs

| The ask | Read (from `references/`) |
| :-- | :-- |
| Look up or edit records — find a lead, full lead detail, contacts, notes, ad-hoc tasks, statuses, curated fields, the worklist | `records.md` |
| Cold outreach cadence — reach-outs & follow-ups due, record a touch, replied / stop chasing, touch history, funnel & outreach-load report | `records.md` + `cold-outreach.md` |
| Live deals — land the discovery call, open / advance / close an opportunity, next steps, won / lost, inbound intake, pipeline & forecast | `records.md` + `deals.md` |
| A reply just came in (the cold → warm seam), or a report spanning funnel *and* pipeline | `records.md` + both flow refs |
| Health audit / cleanup — "what's broken", stuck leads, orphan / duplicate tasks, reconcile | `cleanup.md` |
| Write research findings into the CRM — create/upsert a researched lead with attributes, contacts, the research note; attach a report / audit URL | `records.md` + `intake.md` |
| Draft an email / DM / message for a lead from CRM data | `draft-outreach.md` |

## Rules that apply everywhere

- **Resolve ids at runtime** — `dogfu crm status list` for statuses, `dogfu crm
  opportunity pipelines` for deal stages. Labels are account-specific; never hardcode.
- **Never assume an event.** Close can't see LinkedIn/X/email/calls — only record a
  touch, a reply, a call, or a signature when the user states it happened.
- **Writes change external CRM state.** After any write, confirm in one line what
  changed and what's next. Deletes need an explicit ask.
- **Put data where it belongs** — the placement rules live in `records.md`.
