---
name: berlin-theme
description: >-
  Berlin's brand theme — the editorial cream/paper visual identity to apply whenever you
  design or build any branded visual asset for Berlin. Use this for landing pages, pitch /
  proposal pages, one-pagers, social posters, ad creative, email layouts, decks, PDFs, or
  any HTML/CSS/graphic where the output should look and read like Berlin. It supplies the
  palette, typography, layout, components, motion, and copy voice; the user decides the
  output format and content — this skill only tells you the theme so the result is on-brand.
  Trigger phrases include "make it on-brand", "design a Berlin landing/pitch page", "build a
  sales one-pager", "create a social poster / ad", "style this in our brand", or any request
  to produce a Berlin-branded visual. This is Berlin's house style, distinct from the generic
  `design:*` critique/handoff skills — reach for this when the task is *applying our brand*.
---

# Berlin — Brand Theme

This skill is the **style brief for anything that should look like Berlin**. You bring the
asset the user asked for (an HTML page, a poster, an email, a deck slide, a PDF) — this file
tells you how to make it on-brand. The user decides *what* to build and *what it says*; you
own *how it looks and reads*.

How to use it:

- Read the theme below and apply it faithfully. When in doubt, default to the **Quick
  reference** at the bottom — those six rules carry most of the look.
- It is medium-agnostic. The same palette, type system, and voice apply whether you're
  writing inline-CSS HTML, a social-image layout, or copy for an email.
- This file is the complete and self-contained brief — everything you need is below.

> One-line vibe: **Editorial, print-magazine calm.** Warm paper background, near-black
> ink, a single deep-forest-green accent, big serif display type over clean sans/mono body.
> Confident and understated — never loud, gradient-y, or "tech-startup."

***

## 1. Color palette

Warm paper base + near-black ink + one restrained accent. No pure white, no pure black.

| Role             | Name             | Hex                           | Use                                                   |
| :--------------- | :--------------- | :---------------------------- | :---------------------------------------------------- |
| Background       | Paper            | `#FBF4E6`                     | Default page/canvas background (warm cream)           |
| Background alt   | Paper 2          | `#F3ECDD`                     | Subtle secondary panels, recessed areas               |
| Surface          | Paper card       | `#FFFAF0`                     | Cards, callouts, anything raised off the page         |
| Text             | Ink              | `#0F0F0E`                     | Headlines, primary text (near-black, not #000)        |
| Text             | Ink 2            | `#2B2A26`                     | Body copy, secondary text                             |
| Text             | Ink mute         | `#6B6A64`                     | Captions, labels, metadata, fine print                |
| **Accent**       | **Forest green** | **`#1F4938`**                 | The brand accent — links, emphasis, key numbers, CTAs |
| Accent ink       | Accent ink       | `#FBF4E6`                     | Text/icons placed *on* the green accent (= paper)     |
| Accent soft 2    | Mauve/pink       | `#D8A4D2`                     | Atmospheric background glow only (decorative)         |
| Accent soft 3    | Warm amber       | `#F7C38A`                     | Atmospheric background glow only (decorative)         |
| Warning          | Rust             | `#B4481F`                     | Negative/contrast states ("the agency way"), warnings |
| Inverted surface | Dark             | `#0F0F0E` bg / `#FBF4E6` text | Proof bars, contrast sections, footer                 |
| Live indicator   | Green dot        | `#3AA76D`                     | "Always-on / live" status dot                         |

**Rules / hairlines** (the editorial grid lines that hold layouts together):

* Light rule: `rgba(15,15,14,0.10)` — most dividers, cell borders
* Strong rule: `rgba(15,15,14,0.22)` — emphasized borders, top/bottom of stat rows

**Usage discipline:**

* Green is the *only* real accent. Use it sparingly — one accented element per view wins. Italicize-and-green is the signature "emphasis" move (see typography).
* Mauve/amber are **decorative atmosphere only** (soft blurred radial glows behind hero/feature sections). Never use them for text, fills, or chart bars.
* For inverted/dark sections, flip to dark ink background with paper-colored text.

***

## 2. Typography

Three families, each with a clear job. The serif/sans contrast is the heart of the look.

| Family            | Stack                                         | Role                                                       |
| :---------------- | :-------------------------------------------- | :--------------------------------------------------------- |
| **Display serif** | `Instrument Serif`, Times New Roman, serif    | All headlines, big numbers, the wordmark. Weight 400 only. |
| **Body sans**     | `Geist`, -apple-system, system-ui, sans-serif | Body copy, UI, buttons, most text.                         |
| **Mono**          | `Geist Mono`, ui-monospace, monospace         | Eyebrows, labels, stat keys, tags, metadata.               |

**Headline style:**

* Serif, weight **400** (never bold), tight tracking `-0.02em` to `-0.025em`, line-height `~0.96`.
* Large and confident: hero scales `clamp(44px → 108px)`.
* **Signature emphasis:** set the key phrase in *italic serif + forest green*.
  e.g. "Show Up Right When They're Searching for *What You Solve.*"

**Eyebrows / labels / stat keys:**

* Mono, ~10–11px, `UPPERCASE`, letter-spacing `0.1em–0.16em`, in Ink-mute.
* This is the "editorial kicker" that sits above headlines and labels every stat.

**Body copy:**

* Geist sans, 16–19px, line-height 1.5–1.6, color Ink 2.
* Emphasis within body = bump to Ink + weight 500 (not bold, not colored).

**Big stat / number style:**

* Serif, large (34–44px+), line-height 1, with a small Ink-mute unit beside it.
* Pattern: mono uppercase key on top → big serif value → small mute description below.

***

## 3. Layout & composition

* **Grid-and-rule editorial layout.** Content is organized in clean columns separated
  by thin hairline rules (the `--rule` borders). Think newspaper/spec-sheet, not cards-everywhere.
* **Generous whitespace.** Section padding ~120px desktop / ~72px mobile. Let it breathe.
* **Max content width** ~1240px, centered, 32px side padding (20px mobile).
* **Stat rows**: 4 across, divided by vertical rules, bounded top & bottom by a strong rule.
* **Faint background grid** (80px squares at low opacity, masked to fade at edges) adds
  the "engineering paper" texture behind hero sections.
* **Atmospheric glow**: soft blurred radial gradients (mauve + amber) in a hero corner,
  heavily blurred (~40px) and low opacity. Subtle warmth, never a focal point.

***

## 4. Components & shapes

* **Buttons**: fully rounded pills (`border-radius: 999px`), 14px Geist, weight 500.
  * Primary = solid Ink background, paper text.
  * Secondary = transparent with a strong-rule border, Ink text.
  * Accent = solid green background, paper text.
  * Micro-interaction: scale to 0.98 on press.
* **Pills / tags**: rounded, hairline border, mono uppercase label. Tones: ink, accent (green), solid.
* **Cards / callouts**: Paper-card background, 1px light-rule border, small radius (4–6px),
  very soft shadow. Flat and crisp — not glassy, not heavily shadowed.
* **Corners**: small radii (4–6px) on surfaces; pills (999px) only for buttons/tags.
* **Live dot**: 6px green circle that gently pulses — the "always-on" motif.
* **Wordmark**: "*Berlin*" in italic serif + a mono `/services` suffix in mute.

***

## 5. Motion

* Quiet and quick. Fade-up reveals (translateY ~14px → 0, opacity, ~0.7s ease) as
  sections enter. Staggered delays (80/160/240ms) for hero elements.
* Slow marquees for logo/trust strips (~42s linear loop).
* Button press scale. Pulsing live dot. Always respect `prefers-reduced-motion`.
* No bounce, no flashy transitions. Motion is a whisper, not a show.

***

## 6. Voice & tone (copy)

Matches the visual restraint — plain-spoken, confident, specific, never hypey.

* **Direct and benefit-led.** "Show Up Right When They're Searching for What You Solve."
* **Concrete numbers over adjectives.** "¼ the cost," "80+ signals watched," "24/7,"
  "automating up to 80% of SEO effort." Quantify the claim.
* **Honest, low-pressure CTAs.** "Book a 40-min call" / "Get a free audit instead" /
  "No prep, no commitment." Offer an easy off-ramp, never a hard sell.
* **Define the contrast plainly** (vs. agency / vs. software) without trash-talking.
* **Sentence case** in copy; reserve UPPERCASE for mono eyebrow labels only.
* Avoid: buzzword stacks, exclamation marks, "revolutionary/cutting-edge," emoji.

***

## Quick reference (the 6 things to get right)

1. **Paper** **`#FBF4E6`** **background, Ink** **`#0F0F0E`** **text** — warm, never white-on-black.
2. **Forest green** **`#1F4938`** is the only accent — used sparingly.
3. **Headlines in Instrument Serif, weight 400**, with the key phrase in *italic green*.
4. **Mono UPPERCASE eyebrows/labels** above sections and on every stat.
5. **Thin hairline rules + lots of whitespace** — editorial grid, not card soup.
6. **Quantified, low-pressure copy.** Numbers, calm confidence, an easy off-ramp.

***

## Drop-in CSS variables

A convenience starting point for HTML/CSS assets — mirrors the tokens above. Pull in the
fonts (`Instrument Serif`, `Geist`, `Geist Mono`) however your target supports.

```css
:root {
  --paper: #FBF4E6;
  --paper-2: #F3ECDD;
  --paper-card: #FFFAF0;
  --ink: #0F0F0E;
  --ink-2: #2B2A26;
  --ink-mute: #6B6A64;
  --accent: #1F4938;       /* forest green — the only real accent */
  --accent-ink: #FBF4E6;   /* text on the green */
  --rust: #B4481F;         /* warnings / contrast states */
  --green-dot: #3AA76D;    /* live indicator */
  --glow-mauve: #D8A4D2;   /* decorative atmosphere only */
  --glow-amber: #F7C38A;   /* decorative atmosphere only */
  --rule: rgba(15, 15, 14, 0.10);
  --rule-strong: rgba(15, 15, 14, 0.22);

  --font-serif: "Instrument Serif", "Times New Roman", serif;
  --font-sans: "Geist", -apple-system, system-ui, sans-serif;
  --font-mono: "Geist Mono", ui-monospace, monospace;
}
```
