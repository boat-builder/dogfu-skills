# narrative-audit — skill source

This is the `narrative-audit` skill, shipped inside the **`dogfu`** plugin. It triggers when
someone wants a **brand narrative & category-coherence** check on a domain: do the market, the
brand's own site, its customers, third-party sources, the founder, and live AI answers all tell
the same story about *what category the brand is in and why it belongs*? It reports where the
sources agree, where they diverge, and **why each divergence costs AI selection**.

It is **outside-in** (public data, a domain you don't own) and runs the research + analysis in a
**sub-agent** so the main skill context stays lean. It complements `first-audit` (the full
SEO/AEO prospect audit) and `lead-research` (which qualifies a new target) — both of which can
run the same coherence check *on explicit request* via the shared playbook.

## Files
- `SKILL.md` — the thin orchestration layer: authenticate `dogfu`, optionally seed from the
  Close prior, spawn the coherence sub-agent with the playbook + inputs, deliver its output. The
  YAML frontmatter `name` / `description` controls when the skill triggers.
- `references/brand-narrative-coherence.md` — the **runtime playbook** the sub-agent executes:
  the axes, the canonical-claim schema, panel discovery, tooling classes, the coherence judge,
  and the output contract. Read only by the sub-agent.

## The shared playbook (duplicated by design)
`references/brand-narrative-coherence.md` is the **canonical** copy. Identical copies live in
`first-audit/references/` and `lead-research/references/` so each skill stays self-contained and
portable — the skill protocol resolves references relative to each skill's own directory, and
there is no cross-skill include. **Edit the canonical here, then copy it across by hand:**

```bash
cp dogfu/skills/narrative-audit/references/brand-narrative-coherence.md \
   dogfu/skills/first-audit/references/brand-narrative-coherence.md
cp dogfu/skills/narrative-audit/references/brand-narrative-coherence.md \
   dogfu/skills/lead-research/references/brand-narrative-coherence.md
```

## The `dogfu` CLI dependency
Same as the other dogfu skills: this is the orchestration layer and shells out to the published
`dogfu` CLI (`pip install dogfu`), authenticated through the **dogfu MCP**
(`get_setup_instructions` → `dogfu configure`). It uses the `dogfu` read groups (`google`,
`chatgpt`, `seo`, `linkedin`, `x`, and read-only `crm` for the optional prior) plus native web
reading for pages `dogfu` doesn't model.

## How to edit
1. Edit `references/brand-narrative-coherence.md` here (canonical), then `cp` it to the other two
   skills (above). That manual copy is the sync — there is no build step.
2. To change *when the skill fires*, edit the `description` frontmatter in `SKILL.md`.
3. To change how `first-audit` / `lead-research` invoke the check, edit the short opt-in blocks
   in their `SKILL.md`s — both fire **only on an explicit user request**.

Changes take effect wherever the `dogfu` plugin is installed once you reinstall/update the
plugin from this marketplace (see the repo-root `README.md` for install steps).
