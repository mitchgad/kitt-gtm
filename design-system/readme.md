# Kitt Design System

> **Company:** Zone Blue Pty Ltd
> **Product:** Kitt (lowercase "kitt" in brand usage)
> **Site:** [zoneblue.ai](https://zoneblue.ai) · **App:** [kitt.zoneblue.ai](https://kitt.zoneblue.ai)
> **Source of truth:** github.com/zoneblue — Tailwind config + self-hosted fonts + component patterns
> **Figma:** Zone Blue / Kitt — Brand Reference (attached .fig file)

**Positioning, voice and terminology are governed by `marketing-strategy.md` — that document wins over anything restated here.** The signed brand rubric, verbatim:

> **The four:** **Personal** (you're not a number) · **Connected** (nothing falls through) · **Clear** (never confused) · **Ahead** (never caught out).

kitt is priced at $69 AUD/month per clinician (Professional plan).

Founded by Leigh Kelson, Richard Low, Nathan Carloss, and Freya Simmonds.

---

## CONTENT FUNDAMENTALS

- **Tone:** set by the four brand attributes and the five tone dials (Warmth, Formality, Density, Push, POV) in the strategy doc's VOICE section, which carries a per-context settings table. Do not describe tone from this file.
- **Casing:** Sentence case for headings and UI labels. "Kitt" is always lowercase in brand usage ("kitt") except at the start of sentences.
- **Voice:** second person — "you / your client", never "users". "client" leads as the default; "patient" is permitted but never mixed with "client" inside a single asset.
- **No emoji** in the brand. Clean, text-driven communication.
- **CTAs:** Short, action-oriented. "Start for free", "Get started today", "Choose plan". Verb-first.
- **Descriptions:** concise, one sentence. Lead with the between-visits loop — the two-way record connecting clinician and client. Never lead with scribe or note-taker framing; the note is one input to the loop, never the headline.
- **Contact style:** Friendly but professional. Marketing from "Kitt" (hello@zoneblue.ai), personal from named staff.

---

## VISUAL FOUNDATIONS

### Colors
- **Primary dark:** `#1E0B4B` (kitt-dark) — deep purple, used for hero sections, headings, dark backgrounds
- **Mid-purple:** `#2D1570` (kitt-mid) — accent dark purple
- **Pink:** `#F0408A` (kitt-pink) — primary CTA color, links, accents. The signature action color.
- **Cyan:** `#00C4CC` (kitt-cyan) — secondary CTA, eyebrow text
- **Yellow:** `#FFCD32` (kitt-yellow) — logo dot, small highlights only
- **Purple:** `#9D71F7` (kitt-purple) — accent purple
- **Error only:** `#F15A29` (`--color-error`) — error and validation states only. Orange is **killed as a brand accent**: the strategy doc's BRAND GUIDE reads "pink + purple are the brand, minimise/kill orange". Never use it as a marketing, accent or badge colour.
- **Off-white:** `#F5F3FA` (kitt-off) — light section background, has a subtle lavender tint
- **Grays:** Tailwind gray scale (50–900), with Gray-600 `#4B5563` for body text and Gray-500 `#6B7280` for secondary text

### Typography
- **Outfit** (variable, 300–700): The primary typeface for everything — body text (400), headings (600), buttons and nav CTAs (700). Email fallback: Arial, Helvetica, sans-serif.
- **Instrument Serif** (400 regular + italic): Display/accent — pull quotes, callout blocks, special headings, and hero stat numbers (see Data Viz). Preloaded for LCP. Email fallback: Georgia, Times New Roman, serif.
- Both loaded from Google Fonts (original brand uses self-hosted woff2 from github.com/zoneblue).

### Spacing
- 4px base unit. Scale: 4, 8, 12, 16, 20, 24, 32, 40, 48, 64, 80, 96px.
- No dead space. Unused space is not a design principle — fill surfaces with texture, depth, and content.

### Backgrounds & Surfaces — rich, textured, dark-first
- **Depth over flatness.** Dark hero sections on deep purple, built up with layered aurora glows, mesh and grain — never one flat colour.
- **Light sections:** off-white `#F5F3FA` base. The treatment of light sections is an open decision — not yet signed off.
- **The sanctioned texture system** (see the Textures and Patterns card — every device from brand DNA): aurora, deep mesh, dawn wash, dot texture (fine staggered crisp SVG — never coarse CSS radial-gradient dots), soft blobs, glow orbs, grain gradient, photo-as-texture.
- **Real photography (Noosa, founders, clinics) is a texture** — blurred or scrimmed into the palette. Stock photography never is.
- Layer textures behind type; keep one texture device per surface.

### Borders
- Light surfaces: `rgba(30, 11, 75, 0.06)`
- Dark surfaces: `rgba(255, 255, 255, 0.08)`
- Default/neutral: `#E6E6E6`

### Corner Radii
- Buttons (pill): `9999px` for primary CTAs, `8px` for secondary CTAs
- Cards: `8px` standard, `12px` or `16px` for featured
- Inputs: `8px`

### Shadows
- Purple-tinted shadows using `rgba(30, 11, 75, ...)` rather than pure black. Four levels: sm, md, lg, xl.

### Text Colors
- **On light:** Heading `#1E0B4B`, body `#4B5563`, secondary `#6B7280`, muted `#9CA3AF`
- **On dark:** Heading `#FFFFFF`, body `rgba(255,255,255,0.65)`, muted `rgba(255,255,255,0.40)`

### Hover & Press States
- Hover: opacity drops to ~88%
- Press: `scale(0.97)` transform
- Transition: `0.15s ease` for opacity, `0.1s ease` for transform

### Animation
- `0.15s ease` transitions on interactive elements. No bounces, no spring physics.
- Hero stats count up over ≈1.2s with an eased spring (the KittStat format — see Data Viz).

---

## ICONOGRAPHY

A custom 34-icon system, drawn for Kitt (files: `assets/icons/<name>.svg`; specimen: the Iconography card).

- **Construction:** 24px grid, 2px rounded stroke lifted from the wordmark's soft letterforms. Scales from 76px hero to 16px UI.
- **The accent rule — meaning-led, never decoration:** ink (`#1E0B4B`) draws the structure; **cyan marks where Kitt does the work** (note lines, waveform, active panel); **pink (`#E5197B`) marks the human** (care, alerts needing a person, the hand-off half). Two-tone cyan→pink only where two peer marks form the Kitt-and-clinician pair.
- **On dark:** ink flips to white; cyan and pink accents hold.
- **Inventory:** the clinical loop (9) · product & AI (7) · value (6) · interface (12). Billing/integration and social-channel glyphs deliberately excluded.
- **Signature direction is an open decision** (living/motion icons vs dot-as-origin). This is the workhorse set, synced as the working fallback. Do not invent new glyphs and do not fall back to Lucide — missing concepts get drawn into this system.
- No emoji. No icon fonts.

---

## PRODUCT UI IN MARKETING

The app in marketing is a **recreation, never a raw screenshot** (see the Product UI in Marketing card).

- Rebuild the one moment that does the job from the tokens, on a deep-ink surface darker than the hero purple.
- Frames: minimal dark desktop window (traffic lights, muted title, no browser chrome) or dark rounded phone slab (screen only). One device per frame; overlap desktop + phone to show the loop.
- Crop tight to the single moment; UI text stays legible at final size; bleeding a frame off the artboard is good cropping.
- Fictional patients only, clinically real physio detail, Apple wearables only.
- Never: admin chrome (sidebar nav, data tables, Archive/Logout/Super Admin), browser chrome, real patient data, the old consumer fitness app.

---

## DATA VIZ & CHARTS

Numbers are the story — one hero stat beats a chart; chart only when the shape matters (see the Data Viz and Charts card).

- **Hero stat (KittStat format):** kicker in Outfit 600 uppercase lavender · number in Instrument Serif (teal; pink for the human side) · label in Outfit 600 · thin pill bar for percentages only · ≈1.2s eased count-up in motion.
- **Series palette (fixed order, never cycled):** chart teal `#00969E` → pink `#F0408A` → purple `#9D71F7` → chart orange `#D14E20`; past four, fold into lavender "Other" `#B7ADD6`. Same four on dark and light — machine-validated for colour-blind separation and ≥3:1 contrast on both surfaces. **Chart teal and chart orange are data-viz-only hues, not brand colours** — they exist because the series palette needs validated separation, and never appear outside a chart.
- Bright brand teal and yellow are **not** series colours (they fail the mark-lightness checks); yellow stays a highlight accent.
- Marks: thin bars with 4px rounded data-ends and 2px gaps; 2px lines with an emphasised endpoint; recessive grid; direct-label the point that matters; tabular numerals.
- Ramps: magnitude = one teal ramp light→dark per surface; polarity = teal ↔ chart orange around a neutral midpoint.
- One y-axis, always. Every figure is real — from the truth docs, live analytics, or Mitch. Demo values never ship.

---

## LOGO

- **Wordmark:** "kitt" — a custom lowercase letterform mark (not set in a typeface)
- Letterforms in pink `#E5197B`; the dot on the "i" in cyan `#1AD0FB`; a trailing dot in yellow `#FFCD32`
- Placement: top-left, small (~40–70px wide)
- Works on both light and dark backgrounds
- **SVG source:** `assets/kitt-logo.svg` — use this file, do not recreate the mark in CSS/type
- Note: the logo pink `#E5197B` is the mark's own color; the CTA/UI pink token remains `#F0408A` (kitt-pink)

---

## FILE INDEX

```
├── styles.css                  ← Global CSS entry (imports only)
├── tokens/
│   ├── colors.css              ← Color custom properties
│   ├── typography.css          ← Type scale, weights, families
│   ├── spacing.css             ← Spacing, radii, shadows
│   └── fonts.css               ← @font-face / Google Fonts import
├── assets/
│   ├── kitt-logo.svg           ← The wordmark
│   └── icons/                  ← 34 custom icons (see ICONOGRAPHY)
├── components/
│   ├── core/
│   │   ├── Button.jsx/.d.ts    ← Primary, secondary, ghost, dark-card
│   │   ├── Badge.jsx/.d.ts     ← Status labels and tags
│   │   ├── Card.jsx/.d.ts      ← Light/dark surface containers
│   │   └── core-card.html      ← Component specimen
│   └── forms/
│       ├── Input.jsx/.d.ts     ← Text input with validation
│       └── forms-card.html     ← Component specimen
├── guidelines/
│   ├── colors-*.html           ← Color specimen cards
│   ├── type-*.html             ← Typography specimen cards
│   ├── spacing-*.html          ← Spacing/radius/shadow cards
│   ├── brand-*.html            ← Logo and aesthetic cards
│   ├── textures.html           ← The texture system
│   ├── product-ui.html         ← Product UI in marketing
│   ├── data-viz.html           ← Charts and stats
│   └── iconography.html        ← The icon system
├── sales-kit/                  ← Sales one-pager + 12 deck slides
├── readme.md                   ← This file
└── SKILL.md                    ← Agent skill definition
```

---

## COMPONENTS

| Component | Variants | Location |
|-----------|----------|----------|
| **Button** | primary (pink pill), secondary (cyan rounded), ghost (outlined), on-pink, on-cyan, on-dark (white inverse buttons) | `components/core/` |
| **Badge** | default, pink, cyan, purple, dark, outline × sm/md/lg | `components/core/` |
| **Card** | light, light-alt, dark, dark-mid × padding/radius/shadow | `components/core/` |
| **Input** | label, placeholder, error, helper text, disabled | `components/forms/` |
