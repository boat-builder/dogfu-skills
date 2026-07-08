# crm — skill source

This is the `crm` skill, shipped inside the **`dogfu`** plugin. It is the **single entry
point for everything to and from Agent Berlin's Close CRM** — it replaces the former
`lead-touch` and `crm-cleanup` skills and also owns the CRM reads/writes that used to be
spelled out inside `lead-research` and `first-audit`.

## The design: a tiny router + per-operation references

`SKILL.md` is deliberately minimal — it loads on any CRM ask, so it carries only what
every operation needs (how to run `dogfu`, the lead model in a breath, the CLI-owned-task
rule, the worklist) plus a **routing table** that names the reference file(s) for the
operation in play. All the depth lives in `references/`, loaded per ask:

- `references/records.md` — the shared read/edit surface: the Lead object, finding and
  reading leads, editing contacts / notes / ad-hoc tasks, the curated attribute flags,
  and the "where data belongs" placement rules. The base layer most operations pair with.
- `references/cold-outreach.md` — the cold cadence: the touch model, the system-managed
  outreach fields, `touch record / reply / stop`, the cold queue, funnel report.
- `references/deals.md` — the warm flow: status vs opportunity, the qualification gate,
  the pipeline (Discovery → Trial → Proposal → Won/Lost), `opportunity` verbs, inbound
  intake, pipeline / forecast report.
- `references/cleanup.md` — the read-only health audit: the check catalog (reconcile +
  the drift / readiness / staleness / ownership lenses), severities, and fix routing.
- `references/intake.md` — writing findings into the CRM: the pursue→Qualified /
  drop→Bad Fit contract, the one-pass upsert (lead + attributes + contacts + note), and
  attaching deliverable URLs (e.g. a published first-audit report).
- `references/draft-outreach.md` — composing an email / DM from what the CRM knows about
  a lead: the reads to pull, grounding rules, stage/channel guidance.

Typical loads: a lookup is `SKILL.md + records.md`; recording a touch is `SKILL.md +
records.md + cold-outreach.md`; an audit is `SKILL.md + cleanup.md`. The point is that no
ask ever pays for the whole manual.

## Adding a new CRM operation

This is the extension pattern for future CRM operations: add one self-contained file
under `references/` (state at the top what it assumes from `SKILL.md` / `records.md`),
add one row to the routing table in `SKILL.md`, and — only if the new operation needs new
trigger phrases — extend the frontmatter `description`. Don't grow `SKILL.md` itself:
anything longer than a routing row belongs in the reference.

## How other skills use it

`lead-research` and `first-audit` do their own research/audit work and **hand off to this
skill for the CRM step** — their docs point at `../crm/references/intake.md` (writes) and
`../crm/references/records.md` (reads). Anything CRM-shaped that a future skill needs
should point here the same way rather than restating commands.

## The `dogfu` CLI dependency

Same as the other dogfu skills: the skill is the orchestration layer and shells out to
the published `dogfu` CLI (`pip install dogfu`), installed and authenticated through the
**dogfu MCP** (`get_setup_instructions` → `dogfu configure --otp <OTP> --title "…"`).
CRM (Close) calls go through the backend under the Close API key set once in the admin
**Console → CRM Integration**.

## The cadence lives in the CLI, not here

The touch schedule — between-touch waits, how the next-touch reminder is computed — is
defined in the CLI's config and applied by `dogfu crm touch record`. The `[dogfu:cadence]`
/ `[dogfu:engage]` / `[dogfu:deal:<opp_id>]` tasks are CLI-owned (single writer), with
`dogfu crm reconcile` as the backstop. The references document this so the agent can
reason about it — but the CLI (in the berlin backend repo) is the source of truth. If its
commands or flags change, update `records.md` and the flow references to match.

## How to edit

1. Edit `SKILL.md` or a reference (or ask Claude to).
2. To change *when it fires*, edit the frontmatter `description`. To change *what an
   operation does*, edit its reference. Keep the split honest: `SKILL.md` stays a router;
   anything both flows need goes in `records.md`; anything one flow needs goes in its own
   reference.

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update
the plugin from this marketplace (see the repo-root `README.md` for install steps).
