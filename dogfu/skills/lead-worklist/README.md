# lead-worklist — skill source

This is the `lead-worklist` skill, shipped inside the **`dogfu`** plugin. It triggers when a BDR
wants the **read-only worklist** of leads to work right now — "who do I work on today", "what's on my
plate", "give me the next N leads", or a narrower slice ("just my reach-outs", "follow-ups due",
"leads I'm engaging", "deals that need action") — in **one stage or across all stages**, each row
carrying the contact and the context to act. It is the **cross-phase daily brief** that sits across
the two action skills.

## What it is (and is not)
- **Read-only.** It runs the one queue command `dogfu crm worklist`, presents the result, and hands
  each row off to the skill that records the action. It never writes to the CRM.
- **Cross-phase.** `dogfu crm worklist` reads the real due-task inbox and classifies every row
  (reach-out / follow-up / deal / ad-hoc) in one call — the CLI does the composing, not the skill.
  Pre-gate Engaged leads (replied/inbound, no task yet) are pulled separately from `lead list -s
  <Engaged>` since they aren't tasks. `lead-touch` and `lead-engage` own *recording* actions; this
  skill is the place "what should I work on across everything" lives.
- **Not an action skill.** To record a touch use **lead-touch**; to move a deal use **lead-engage**;
  for a health/anomaly audit use **crm-cleanup**; to research a new target use **lead-research**.

## Files
- `SKILL.md` — the operating manual: the one `crm worklist` queue command, the
  **tiered-context model** (Layer 1 act-on-it context inline for every row; Layer 2 deep context
  pulled on demand only for the lead being actioned — agents are not forced to read it for the whole
  queue), how to run `dogfu` read-only, the read-only command surface, and the operating rules. The
  YAML frontmatter `name` / `description` controls when the skill triggers.

There are no `references/` or `icps/` here — like `lead-touch`, this skill is a single self-contained
file.

## The `dogfu` CLI dependency
Same dependency as the other CRM skills: the skill is the orchestration layer and shells out to the
`dogfu` CLI for all reads. The skill **does not bundle `dogfu`** — it's a published package (`pip
install dogfu`) installed and authenticated through the **dogfu MCP** (`get_setup_instructions` →
`dogfu configure --otp <OTP> --title "…"`). CRM (Close) calls go through the backend under the Close
API key set once in the admin **Console → CRM Integration**. See the **Running `dogfu`** section of
`SKILL.md`.

## How to edit
1. Edit `SKILL.md` (or ask Claude to).
2. To change *what it does*, edit the body. To change *when it fires*, edit the `description` in the
   frontmatter — keep it disambiguated from `lead-touch`/`lead-engage` (which own *recording*
   actions) and `crm-cleanup` (which owns the *health audit*).
3. Keep it tight, self-contained, and **read-only** — this skill must never be the thing that writes.

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update the plugin
from this marketplace (see the repo-root `README.md` for install steps).
