> **Static port.** In the original (agentberlin) flow this guide was fetched live with
> `agentberlin report design`. The dogfu publish path has no live design endpoint, so the
> authoritative guide is **vendored here** and you build the report against it directly —
> follow it exactly. The only thing that can drift is the vendor chart scripts' version
> (see *Data-driven charts*); everything else is stable. (Source of truth upstream:
> `backend/internal/reports/design_guide.md`.)

---

# Design guide (authoritative — follow exactly)

This is the complete recipe book for styling every element you're asked to render. Your brief tells you WHAT to render (the data shape, sometimes a component type like "card" or "chart"); this guide decides HOW it looks — every color, font, size, weight, padding, radius, and animation is decided here. Do not invent new tokens, gradients, shadows, or animations beyond this guide.

## Root variables

Declare every color and font family as a CSS custom property on `:root` and reference them by name everywhere. Do not hardcode hex values or font names elsewhere in the stylesheet.

Colors (identical to the workspace dashboard palette):

| Token         | Value    | Use                                               |
| ------------- | -------- | ------------------------------------------------- |
| --bg          | #f6f5f2  | Page canvas                                       |
| --surface     | #ffffff  | Card background                                   |
| --surface-2   | #faf9f7  | Nested / inset areas, neutral pill fills          |
| --border      | #ece9e2  | Default card border, row dividers                 |
| --border-2    | #e2ded5  | Emphasized / button borders                       |
| --ink         | #141414  | Primary text, titles                              |
| --ink-2       | #3d3d3d  | Body text, meta-row values                        |
| --ink-3       | #6f6d68  | Secondary text, mono labels, captions             |
| --ink-4       | #a8a59e  | Tertiary, placeholders, footer line               |
| --accent      | #7c4dff  | Signature purple                                  |
| --accent-2    | #5b2fd1  | Accent text (italic-serif), gradient end, links   |
| --accent-soft | #f3efff  | Accent pill backgrounds                           |
| --accent-line | #d9ccff  | Accent borders                                    |
| --ok          | #1f9d55  | Done states, upward trends, live indicators       |
| --ok-soft     | #e6f6ed  | Live-chip / success pill backgrounds              |
| --warn        | #b4651a  | Warning states                                    |
| --warn-soft   | #fdf3e2  | Warning pill backgrounds, warning-tinted cards    |
| --danger      | #c2410c  | Critical states, negative deltas                  |
| --danger-soft | #fcebe1  | Critical pill backgrounds, critical-tinted cards  |

Font-family stacks:

    --serif: 'Instrument Serif', 'Iowan Old Style', Georgia, serif;
    --sans:  'Inter', -apple-system, system-ui, sans-serif;
    --mono:  'IBM Plex Mono', ui-monospace, monospace;

## Font loading

Load the named fonts from Google Fonts with exactly this `<head>` block:

    <link rel="preconnect" href="https://fonts.googleapis.com">
    <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&family=Instrument+Serif:ital@0;1&family=IBM+Plex+Mono:wght@400;500;600&display=swap" rel="stylesheet">

These are the only weights and italic variants the design uses — do not request others.

The only external resources a report may load are these fonts and the two charting scripts described under **Data-driven charts (ECharts)** below — nothing else.

## Page defaults

Begin every page with these baseline rules:

    * { box-sizing: border-box; }
    html, body { margin: 0; padding: 0; }
    body {
      background: var(--bg);
      color: var(--ink);
      font-family: var(--sans);
      letter-spacing: -0.005em;
      font-size: 14px;
      line-height: 1.5;
    }

## Typography — three families, three jobs

Within any single element (a card, a row, a header) you will usually mix all three families. The rule is:

- Serif (`var(--serif)`) — DISPLAY ONLY: page title, section headings, card / item titles, large numbers (scores, stats, dial center, pillar values). Always font-weight 400 — the display quality comes from the typeface, not from bolder weights. Never use serif for body copy, labels, or buttons.
- Sans (`var(--sans)`) — body text, descriptions, paragraph copy, button labels, inline `<b>` emphasis. Letter-spacing -0.005em inherited from body.
- Mono (`var(--mono)`) — every label, chip, pill, timestamp, caption, numeric ID, table header, ALL-CAPS text. If it's a short fixed label or a metadata value shown in uppercase, it's mono.

Size anchors:

| Role                                   | Family | Size       | Weight | Letter-spacing |
| -------------------------------------- | ------ | ---------- | ------ | -------------- |
| Page title (h1)                        | serif  | 46px       | 400    | -0.02em        |
| Section heading (h3)                   | serif  | 28px       | 400    | -0.015em       |
| Secondary heading (h2 inside hero)     | serif  | 28px       | 400    | -0.015em       |
| Italic subtitle paragraph              | serif  | 20px ital. | 400    | —              |
| Card title / finding title             | serif  | 20px       | 400    | -0.01em        |
| Card body / description                | sans   | 13.5–14.5px| 400    | —              |
| Body paragraph                         | sans   | 14px       | 400    | -0.005em       |
| Inline emphasis `<b>`                  | sans   | inherit    | 600    | —              |
| Button label                           | sans   | 12–13px    | 500    | —              |
| Meta-row value                         | mono   | 11.5px     | 400    | —              |
| Pill / chip / tag text                 | mono   | 10.5–11.5px| 600    | 0.04–0.1em     |
| Uppercase metadata label               | mono   | 10.5–11px  | 400–600| 0.06–0.1em     |
| Hero score number                      | serif  | 72px       | 400    | -0.03em        |
| Pillar / stat value                    | serif  | 28px       | 400    | —              |
| Footer line                            | mono   | 11px       | 400    | —              |

Minimum rendered body size: 12px. Never go smaller.

## Two-colored headings (italic-serif accent)

Every page-level serif heading uses the signature two-tone motif: most words render in `--ink`, and a short accent phrase inside `<em>` renders italic with `color: var(--accent-2)`. One accent phrase per heading — at most. Examples of the pattern (not the copy — the copy is the brief's):

    <h1>Audit <em>report</em></h1>
    <h3>What Berlin <em>found</em></h3>
    <h2>Your site is in <em>decent shape</em>, improving steadily.</h2>

CSS:

    h1, h2, h3 { font-family: var(--serif); font-weight: 400; margin: 0; color: var(--ink); }
    h1 { font-size: 46px; letter-spacing: -0.02em; line-height: 1; }
    h2 { font-size: 28px; letter-spacing: -0.015em; }
    h3 { font-size: 28px; letter-spacing: -0.015em; }
    h1 em, h2 em, h3 em { font-style: italic; color: var(--accent-2); }

Card / item titles (20px serif) do NOT use the accent — they are plain `--ink`.

An italic-serif paragraph is a different element (full paragraph italic, `--ink-2`, serif 20px, max-width ~780px) used as a page subtitle under h1. No accent phrase inside it.

## Section header row

A section header is the h3 plus a right-aligned mono caption (e.g. "7 issues · prioritised by impact", "last 48 hours", "automated · you don't need to do anything"):

    display: flex; align-items: baseline; justify-content: space-between;
    margin: 36px 0 14px;

    .section-h h3     { font-family: var(--serif); font-size: 28px; font-weight: 400; letter-spacing: -0.015em; margin: 0; }
    .section-h .note  { font-family: var(--mono); font-size: 11px; color: var(--ink-3); }

## Cards — three flavors

All cards share: background `var(--surface)` by default, 1px solid `var(--border)`, border-radius in the 12–18px range. No shadows on ordinary cards.

1. Plain card — border 1px solid `var(--border)`, border-radius 12px, padding 14–22px, no shadow. Used for timeline rows' container, info cards, next-check cards.
2. Hero card — the single largest card at the top of the page. Adds `box-shadow 0 1px 2px rgba(20,20,40,0.03)` and radius 18px with padding `28px 30px`. Only ever one hero card per page.
3. Severity-tinted card — a list card with a soft top-down wash indicating state. Same border-radius 12px, padding `18px 22px`. Tint recipes:

        .card.crit { border-color: #f0d1c5; background: linear-gradient(180deg, #fdf5f0 0%, var(--surface) 40%); }
        .card.warn { border-color: #f0e0c4; background: linear-gradient(180deg, #fdf8ef 0%, var(--surface) 40%); }
        .card.info { background: var(--surface); border-color: var(--border); } /* no tint */

The tint fades from a soft-color top (near `--danger-soft` or `--warn-soft`) into the plain surface by 40% down, so the severity is a hint, not a shout.

## Badges, pills, and chips

All pill-shaped elements share: `display inline-flex`, `align-items center`, `gap 6–7px`, `border-radius 999px`, mono font family, weight 600. Size, colors, and padding vary by role.

- Live chip (signals a live / real-time surface, sits in the header):

        background: var(--ok-soft); color: var(--ok);
        font-family: var(--mono); font-size: 11.5px; font-weight: 600;
        letter-spacing: 0.02em; padding: 5px 11px 5px 9px; border-radius: 999px;
        /* contains a 6px round dot in var(--ok) running the pulse animation */

- Severity tag (inside a card's label row, matches the tint):

        font-family: var(--mono); font-size: 10.5px; font-weight: 600;
        text-transform: uppercase; letter-spacing: 0.08em;
        padding: 2px 7px; border-radius: 999px;
        /* critical: color var(--danger) on var(--danger-soft)
           warning:  color var(--warn)   on var(--warn-soft)
           info:     color var(--ink-2)  on var(--surface-2) */

- Action / status pill (what's happening to this item):

        font-family: var(--mono); font-size: 10.5px; font-weight: 600;
        letter-spacing: 0.04em; padding: 3px 9px; border-radius: 999px;
        /* working/running: color var(--accent-2) on var(--accent-soft), include a 6px pulsing dot in var(--accent)
           neutral / awaiting: color var(--ink-3) on var(--surface-2) */

- Uppercase metadata label (not a pill — just a label next to a pill or above a value):

        font-family: var(--mono); font-size: 10.5–11px; color: var(--ink-3);
        text-transform: uppercase; letter-spacing: 0.08–0.1em;

## Buttons

- Card action button (the default "do something" button inside list/finding cards — ghost-style):

        font-family: var(--sans); font-size: 12px; font-weight: 500;
        color: var(--ink-2); background: var(--surface);
        border: 1px solid var(--border-2); border-radius: 8px;
        padding: 6px 11px;
        /* hover: background var(--surface-2); color var(--ink); */

- Primary dark button (page-level "New task" style, dark):

        font-family: var(--sans); font-size: 12–13px; font-weight: 500;
        color: #ffffff; background: #18181b; /* near-black */
        border: none; border-radius: 10px;
        padding: 8px 12px;
        /* hover: background #27272a */

- Accent pill button (header-level "View report" style):

        font-family: var(--sans); font-size: 12px; font-weight: 600;
        color: var(--accent-2); background: var(--accent-soft);
        border: 1px solid var(--accent-line); border-radius: 10px;
        padding: 8px 12px;
        /* optionally prefixed with a 6px --accent dot with a soft halo: box-shadow: 0 0 0 3px rgba(124,77,255,0.18); */

No other button styles. Pick the closest of these three based on what the brief asks the button to do.

## Meta rows (key–value pairs)

A horizontal strip of small fact pairs at the bottom of a card — e.g. "Pages affected: 47 · First seen: Apr 14 · Impact: indexing":

    display: flex; gap: 18px; margin-top: 10px;
    font-family: var(--mono); font-size: 11.5px; color: var(--ink-3);
    /* each pair: <span><b>Label</b> value</span>
       b tags inside inherit mono, color var(--ink-2), font-weight 600 */

## List rows / timeline rows

A generic row pattern for any sequential list with a time/dot/content/status layout:

    display: grid;
    grid-template-columns: 110px 20px 1fr 130px;
    gap: 14px; align-items: center;
    padding: 10px 18px;
    border-bottom: 1px solid var(--border);
    font-size: 13.5px;
    /* last-child: no border-bottom */

    .t   { font-family: var(--mono); font-size: 11.5px; color: var(--ink-3); } /* timestamp */
    .dot { width: 8px; height: 8px; border-radius: 50%; margin: 0 auto; }
    .name { font-weight: 500; color: var(--ink); }            /* row title */
    .out  { font-size: 12px; color: var(--ink-3); margin-top: 1px; }  /* one-line output */
    .st   { font-family: var(--mono); font-size: 10.5px; text-align: right;
            text-transform: uppercase; letter-spacing: 0.06em; }

    /* dot + status colors by state:
       running: .dot var(--accent) + pulse animation; .st color var(--accent-2)
       done:    .dot var(--ok);                         .st color var(--ok)
       awaiting: .dot var(--ink-3);                    .st color var(--ink-3) ("Awaiting you") */

## Grid of small info cards

A 2-column grid for "upcoming / scheduled / next" style content, or any flat list of small labeled items:

    display: grid; grid-template-columns: repeat(2, 1fr); gap: 12px;

    .item         { background: var(--surface); border: 1px solid var(--border); border-radius: 12px; padding: 14px 16px; }
    .item .when   { font-family: var(--mono); font-size: 11px; color: var(--ink-3); text-transform: uppercase; letter-spacing: 0.08em; }
    .item .n      { font-family: var(--sans); font-size: 14.5px; font-weight: 600; margin-top: 4px; color: var(--ink); }
    .item .d      { font-family: var(--sans); font-size: 12.5px; color: var(--ink-3); margin-top: 2px; }

## Charts and progress indicators

Two rendering paths, chosen by complexity:

- **Single-value indicators** (a score dial, a lone progress bar) — hand-authored inline SVG, using the two primitives below. No library; they are trivial and weightless.
- **Data-driven charts** (anything backed by a series of data: lines, bars, areas, pies, heatmaps, gauges over real numbers) — Apache ECharts via the registered `berlin` theme, described under **Data-driven charts (ECharts)** at the end of this section. Do not hand-roll multi-point charts in raw SVG.

1. Radial arc (score dial) — a circular progress arc. Outer container 220x220 (scale proportionally for other sizes). The SVG group is rotated -90deg so progress starts at 12 o'clock:

        <svg width="220" height="220" viewBox="0 0 220 220" style="transform: rotate(-90deg)">
          <circle cx="110" cy="110" r="92" stroke="var(--border)" stroke-width="14" fill="none"/>
          <circle cx="110" cy="110" r="92" stroke="url(#g1)" stroke-width="14" fill="none"
                  stroke-linecap="round"
                  stroke-dasharray="578" stroke-dashoffset="<578 * (1 - value/100)>"/>
          <defs>
            <linearGradient id="g1" x1="0" y1="0" x2="1" y2="1">
              <stop offset="0%"   stop-color="#7c4dff"/>
              <stop offset="100%" stop-color="#5b2fd1"/>
            </linearGradient>
          </defs>
        </svg>

    Center of the arc holds a serif 72px number, a mono 10.5px uppercase label below it (letter-spacing 0.1em, color `var(--ink-3)`), and an optional 12px weight-600 delta row colored `var(--ok)` for positive or `var(--danger)` for negative.

2. Thin progress bar — 4px high, used inside small stat cards:

        height: 4px; background: var(--border); border-radius: 2px; overflow: hidden;
        /* fill: */
        height: 100%; width: <percent>%;
        background: linear-gradient(90deg, var(--accent), var(--accent-2)); border-radius: 2px;

For multi-value stat tiles (used with the progress bar) the container is:

    border: 1px solid var(--border); border-radius: 10px; padding: 12px 14px;
    background: var(--surface-2);
    /* label (mono 10.5px uppercase, letter-spacing 0.08em, --ink-3)
       value (serif 28px, with smaller trailing "/100" in --ink-4 at 14px)
       delta row (11.5px, var(--ok) for up with ↑, var(--danger) for down with ↓)
       progress bar (as above) */

### Data-driven charts (ECharts)

Charts backed by a data series use Apache ECharts with the registered `berlin` theme, which owns all styling (palette, fonts, gridlines, legend, tooltips) — you pick the chart type and pass data. Any type works: line, bar, area, pie/donut, scatter, gauge, heatmap, radar, sankey, …

Load both scripts once; the page's `:root` tokens and `echarts.min.js` must come before `theme.js` (the theme reads tokens from `:root` at load):

    <script src="https://files.agentberlin.ai/vendor/echarts/5.6.0/echarts.min.js"></script>
    <script src="https://files.agentberlin.ai/vendor/berlin-theme/v1/theme.js"></script>

Init with the SVG renderer and the theme, then set type + data only:

    echarts.init(el, 'berlin', { renderer: 'svg' }).setOption({ /* type + data */ });

Gradient helpers (so you never hand-roll them): `BERLIN.accentGradient('h'|'v')` for the filled-purple emphasis (bar fills, gauge arc), `BERLIN.fadeGradient()` for the area wash under a line, `BERLIN.tokens` for raw hex.

Rules: always `renderer:'svg'` and the `'berlin'` theme (never canvas/default); stay on the token palette; each chart sits in a standard card with an explicit container height (e.g. 300px — ECharts won't render at zero height); don't override the theme's axes/gridlines/tooltips; no 3D/GL or chartjunk. A data-driven score uses a `gauge`; the inline-SVG dial stays only for a static hero score.

## Motion and accent primitives

These are the only non-solid fills and the only animation in the system:

- Accent gradient — `linear-gradient(135deg, var(--accent), var(--accent-2))` — the one filled-purple treatment. Use it for brand marks (logo tile), dial / progress-arc stroke, and filled progress-bar fills (the bar variant is 90deg). No other gradients anywhere in the page chrome — inside an ECharts chart the equivalent emphasis comes from `BERLIN.accentGradient` / `BERLIN.fadeGradient` (see Data-driven charts), not new CSS gradients.
- Live-pulse halo — a `box-shadow` that expands and fades, applied to a small circle to signal something is live, running, or animating. Always color-keyed to state. Exact keyframe in the `--ok` flavor:

        @keyframes pulse {
          0%   { box-shadow: 0 0 0 0   rgba(31, 157, 85, 0.45); }
          70%  { box-shadow: 0 0 0 8px rgba(31, 157, 85, 0);    }
          100% { box-shadow: 0 0 0 0   rgba(31, 157, 85, 0);    }
        }

    For the `--accent` (running/working) variant use `rgba(124, 77, 255, …)`; for `--danger` (critical) use `rgba(194, 65, 12, …)`. Duration 1.8s, iteration infinite, target element a 6–10px circle.

No other shadows, no other animations unless the brief explicitly asks for one.

## Brand mark / logo tile

A 40x40 rounded-square tile carrying a single letter (typically "b"):

    width: 40px; height: 40px; border-radius: 10px;
    background: linear-gradient(135deg, var(--accent), var(--accent-2));
    color: #ffffff; display: grid; place-items: center;
    font-family: var(--sans); font-size: 18px; font-weight: 700; letter-spacing: -0.02em;
    box-shadow: 0 2px 4px rgba(91, 47, 209, 0.25), inset 0 1px 0 rgba(255, 255, 255, 0.2);

Next to it (when the page header identifies "Berlin for <client>"), stack two lines: top line mono 11px uppercase `--ink-3` letter-spacing 0.08em ("BERLIN · FOR"); bottom line serif 22px letter-spacing -0.01em (the client name).

## Footer

Report footer is a single divider + one line of mono text, split left/right:

    margin-top: 48px; padding-top: 20px;
    border-top: 1px solid var(--border);
    display: flex; align-items: center; justify-content: space-between;
    color: var(--ink-4); font-family: var(--mono); font-size: 11px;
    /* inline links inside: color: var(--accent-2); text-decoration: none; hover underlines */

## Corners (border-radius) scale

Only these radii exist:

- 999px — pills, chips, round dots
- 18px — hero card
- 14px — list container with many rows (e.g. timeline wrapper)
- 12px — standard cards, severity-tinted cards
- 10px — stat tiles, logo tile, primary / accent buttons
- 8px   — action buttons
- 4px   — progress-bar segments
- 2px   — very small bar fills

Never use 6px, 16px, 20px, or any radius outside the scale above.

## Layout and wrap

Page wrapper:

    max-width: 1120px; margin: 0 auto; padding: 32px 36px 80px;

Section gaps: 36px above an h3 section header; 14px below it to the first element. Cards in a vertical stack gap 12px. Meta-row gap between pairs 18px.

## Responsive

Single breakpoint at 820px. Below it:

    .wrap { padding: 20px 18px 60px; }
    h1    { font-size: 32px; width: 100%; }
    /* hero grid: single column */
    /* 4-up stat tiles: 2 columns */
    /* list/timeline rows: drop the right-aligned status column */
    /* list cards with an action column: action stacks below the body */
    /* 2-col grid of info cards: 1 column */

## Discipline

- Do not invent colors, font families, radii, gradients, shadows, animations, or spacing values outside this document.
- Do not use drop shadows on ordinary cards; do not introduce new gradients.
- No decorative illustrations, background textures, or emoji in the chrome.
- No marketing language, and no CTAs to buy or upgrade anything. **(One deliberate exception for this skill: the single book-a-call CTA at the very end of the audit — see `report.md`. Nothing else on the page is promotional.)**
- When copy is baked into the template (footer line, empty states), use plain language and refer to Berlin in third person.
- When you mix type within one element, follow the rule: serif for titles/large numbers, sans for body/buttons, mono for labels/chips/timestamps — never swap them.
