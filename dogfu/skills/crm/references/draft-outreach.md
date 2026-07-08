# Draft outreach — compose an email / DM / message from CRM data

For "write an email to <lead>", "draft a LinkedIn DM for X", "what should I say to them?".
This flow is **read-only against the CRM**: you pull everything saved about the lead, then
write the message from it. The BDR sends by hand; recording what was sent afterwards is
the cold flow's job (`cold-outreach.md`) — never record a touch for a draft.

***

## Pull the full picture first (concurrent reads)

| What it gives you | Command |
| :-- | :-- |
| Status, description, curated attributes (industry, size, revenue, SEO footprint, momentum…) + embedded contacts | `dogfu crm lead get <lead_id>` (find the id via `lead list -n "<name>"` / `-q "<domain>"`) |
| The people — names, titles, LinkedIn/X urls, emails | `dogfu crm contact list <lead_id>` |
| The saved depth — research write-up, **DM hooks**, decision reason, competitive ratios, published audit/report URLs | `dogfu crm note list <lead_id>` |
| What's already been sent — each touch's number, date, channel, detail | `dogfu crm touch history <lead_id>` |

If the ask names a person, match them in `contacts[]`; if the record has several, ask
which (or default to the decision-maker named in the research note).

***

## Compose from what's saved — never fabricate

- **Ground every personalized line in a stored fact** — the note's DM hooks, a curated
  attribute, a top page or gap from the research. No invented numbers, events, or
  compliments. If you use a fact, it should be traceable to the record.
- **The research note is the seed.** lead-research writes DM hooks (personal, company,
  relevance-bridge) and the competitive read there — start from those, freshen the
  wording, don't re-research.
- **A published audit URL is the strongest give** — if a note carries one, offer it as
  the value ("we ran your site through our audit — here's what it found") rather than
  pitching cold.
- **Match the stage and channel:**
  - *Cold reach-out* — short, hook-first, no pitch-dump; a LinkedIn DM is 2–4 sentences.
  - *Follow-up* — a new angle or a new channel, not a repeat; check `touch history` /
    `touch_channel` for what's been said and which channels are untried.
  - *Warm (Connected / Engaged)* — reference the actual conversation and drive to the
    next concrete step (the discovery call, the deal's next step).
- **Berlin's one-liner** when the message needs a pitch: an AI SEO/AEO platform that
  redirects budget the company **already spends** on SEO / content / agencies — not a new
  line item. For a **bigger brand**, lead instead with the **founder-led consultancy**
  (standing up AI-marketing infrastructure with their existing team, broader than SEO/AEO)
  rather than the self-serve product — pitch the fit the lead's size points to.
- **Voice:** plain, specific, short. Their language (from the note), not marketing-speak.

**If the record is thin** — no note, no hooks, no contacts — say so and offer a
lead-research pass instead of padding the draft with generic flattery.

***

## Output & follow-through

Return the draft (or 2 variants if the ask is open-ended) with a one-line note of which
stored facts it leans on, plus the recipient's handle/email from the contact record so
it's copy-paste sendable. If the BDR then says they sent it, record it — a touch via
`cold-outreach.md` for a cold lead, a note for a warm one.
