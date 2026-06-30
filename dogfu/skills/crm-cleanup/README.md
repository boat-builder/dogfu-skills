# crm-cleanup — skill source

This is the `crm-cleanup` skill, shipped inside the **`dogfu`** plugin. It triggers when
someone wants a **read-only health audit** of the Close CRM — to find leads and tasks that
have drifted out of the regular outreach flow: orphaned or duplicate cadence reminders, leads
still chased after they replied or went Bad Fit, Qualified leads with no contacts / no research
/ no website, duplicate leads, badly overdue follow-ups, unassigned tasks. It **reports** each
anomaly with the exact fix command; it never writes to Close.

It's the audit counterpart to `lead-touch` (which *operates* the flow) and runs after
`lead-research` (which *creates* leads). Use `lead-touch` to apply per-lead fixes;
`dogfu crm touch reconcile --apply` applies the bulk cadence-task repairs.

## Files
- `SKILL.md` — the operating manual: the one-breath model recap, the gather-once read plan,
  the full check catalog (grouped A–E, each with how to detect it, severity, and the fix), the
  honest scope/limits, and the report format. The YAML frontmatter `name` / `description`
  controls when the skill triggers.

Single self-contained file — no `references/` or `icps/`. The check logic is inline so the
agent rarely needs `dogfu --help`.

## The `dogfu` CLI dependency
Same as the other dogfu skills: the skill is the orchestration layer and shells out to the
published `dogfu` CLI (`pip install dogfu`) for all reads, authenticated through the **dogfu
MCP** (`get_setup_instructions` → `dogfu configure --otp <OTP> --title "…"`). CRM (Close) calls
go through the backend under the Close API key set once in the admin **Console → CRM
Integration**. This skill uses only **read** endpoints (`status list`, `lead list/get`,
`task list`, `note list`, `contact list`, `whoami`, and `touch reconcile` without `--apply`,
which is read-only).

## Read-only by design
The skill detects and reports; it does not mutate Close. The cadence-task / deal-task invariant
is delegated to `dogfu crm touch reconcile` (run read-only, repaired with `--apply`) so the CLI
stays the single writer of the outreach state; the skill adds the status / readiness / staleness
/ ownership checks the CLI doesn't compute — see the **Scope & limits** section of `SKILL.md`.

## How to edit
1. Edit `SKILL.md` (or ask Claude to).
2. To change *what it detects*, edit the check tables. To change *when it fires*, edit the
   `description` in the frontmatter.
3. When the CLI gains new read fields (e.g. a created date), update the "Scope & limits" and
   fix columns to match. Reach-out tasks and opportunities have shipped and their checks are on;
   keep the cadence-task section delegating to `reconcile` rather than re-deriving its logic.

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update the
plugin from this marketplace (see the repo-root `README.md` for install steps).
