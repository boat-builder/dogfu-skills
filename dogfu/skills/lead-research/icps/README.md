# ICPs

This folder holds the Ideal Customer Profiles the `lead-research` skill qualifies against.
**One markdown file per ICP.** The skill loads an ICP from here automatically, so you no
longer paste it into every request.

## How an ICP file looks

Each ICP is a markdown file with frontmatter + a body:

```markdown
---
name: saas-seo             # short slug, used to match "use the saas-seo ICP"
label: B2B SaaS doing its own SEO   # human-readable name
default: true              # exactly ONE file should set this
---

<the ICP itself: who we are, who we target, the qualifying criteria,
the disqualifiers, the decision-maker, geography/size — whatever the
skill should judge a target against>
```

## How the skill picks an ICP (at run time)

1. **Caller override** — an ICP pasted inline or a file path given in the request wins.
2. **Named ICP** — if the request names one ("use the enterprise ICP"), the matching
   `name`/`label` file is used.
3. **Default** — otherwise the file with `default: true` is used (or the only ICP, if
   there's just one).

## Adding more ICPs

Drop in another `*.md` file with its own `name`/`label`. Keep `default: true` on exactly
one of them. Then reinstall/update the `dogfu` plugin so the new ICP ships with it.

## Notes

- `README.md` and any file starting with `_` (e.g. `_TEMPLATE.md`) are **scaffolding, not
  ICPs** — the skill ignores them.
- This is a **public** repo. Don't put anything in an ICP you wouldn't want public
  (pricing secrets, named accounts, internal-only commentary).
