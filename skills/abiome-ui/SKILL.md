---
name: abiome-ui
description: "Apply the abiome brand and design system to abiome-org projects and abiome.org properties, using the canonical landing-repository guidelines and @abiome/ds. Do not apply outside that brand."
license: MIT
metadata:
  version: "1.0.0"
  canonical: https://github.com/abiome-org/landing/blob/main/GUIDELINES.md
---

# abiome UI/UX & Brand System

This skill is the authoritative design system and operational standard for all **abiome** digital surfaces: public landing pages, research papers, slide decks, model cards, consoles, and interactive applications under `abiome-org/*` and `*.abiome.org` (e.g., `iso.abiome.org`, AbCP, platform-main).

**Scope constraint:** Never apply this skill, its color tokens, its typography, or its motifs outside the abiome brand boundary.

---

## 1. Canonical Source & Priority

The canonical source of truth for the abiome brand is maintained in the `abiome-org/landing` repository:
- Primary brand guidelines: `GUIDELINES.md`
- Design system documentation: `ds/docs/design-system.md`
- Implementation conventions: `.design-sync/conventions.md`
- Component library: `@abiome/ds` (located in `abiome-org/landing` under `ds/`)
- Core brand assets: `assets/abiome_mark.svg`, `assets/abiome_logo.svg`, `assets/abiome_favicon.ico`

If this skill and `GUIDELINES.md` in `abiome-org/landing` ever drift, treat the repository `GUIDELINES.md` as canonical until both documents are reconciled.

---

## 2. Core Metaphor: The Two Layers

Every abiome surface is structured around two distinct spatial planes:

1. **The Field (Background):** A deep, dithered sea-glass fluid simulation rendered on a full-viewport WebGL canvas. It represents organic living computation, flow, and texture.
2. **The Record / Mosaic (Foreground):** Crisp, opaque white paper panels placed above the field. Typeset with classical editorial rigor, containing all legible text, data tables, and interactive controls.

The relationship between the layers is absolute: text and interactive elements live on white paper; the living field breathes behind and between the mosaic tiles.

---

## 3. Brand Tokens

### Color Palette

| Token | Hex Value | Purpose |
|---|---|---|
| `--ink` | `#1d1b18` | Primary text, titles, deep borders, high contrast elements |
| `--ink-soft` | `#4d463c` | Secondary text, navigation links, explanatory notes |
| `--ink-faint` | `#837a6c` | Muted labels, dotted leader lines, metadata, captions |
| `--paper` | `#ffffff` | Panel surfaces, card backgrounds, modal records |
| `--brand` | `#006e59` | Primary deep green accent, active states, key brand highlights |
| `--field` | `#a8d0bd` | Solid sea-glass fallback color for the background field |
| `--on-field` | `rgba(9, 55, 43, 0.68)` | Contrast ink for rare text directly overlaid on the field |
| `--line` | `rgba(29, 27, 24, 0.16)` | Subtle hairline rules and panel separators |
| `--seaglass` | `#a5d6c2` | Secondary field tint, cycling animation step |
| `--mint` | `#cfe9db` | Highlight selections (`::selection`), active pill fills |
| `--butter` | `#f0dfa2` | Sparse warm highlight wisp in shader field and accents |

### Typography

- **Primary Typeface:** `Alegreya` (weights: 400, 500, 600, 700; italic: 400, 500). Fallback: `"Iowan Old Style", Georgia, serif`.
- **Small Caps:** `Alegreya SC` (weights: 400, 500) for section overlines, table headers, and category tags.
- **Monospace / Code:** Clean tabular monospace (e.g., `JetBrains Mono`, `Fira Code`, `ui-monospace`) for raw sequence data, code blocks, and execution logs.
- **Brand Name Rule:** Always typeset **abiome** in lowercase. Never capitalize as "Abiome" or "ABIOME" in prose, titles, or headers.

### Layout, Spacing & Rhythm

```css
:root {
  --measure: 72rem;
  --rhythm: 1.5rem;
  --gutter: clamp(1rem, 4vw, 3rem);
  --gap: 0.6rem;
  --ease-out: cubic-bezier(0.23, 1, 0.32, 1);
  --ease-in-out: cubic-bezier(0.77, 0, 0.175, 1);
}
```

---

## 4. Hard Brand Rules (The Never List)

- **Zero border-radius (`border-radius: 0`):** Never round corners on panels, buttons, inputs, tabs, or badges. Every card, tile, and record is razor-sharp.
- **No drop shadows:** Never use `box-shadow` to elevate elements. Hierarchy is achieved through contrast, spacing, grid arrangement, and 1px borders.
- **No pure black:** Never use `#000000`. Use `--ink` (`#1d1b18`).
- **No gradient cards:** Never use purple/indigo gradients or mesh fills on content cards.
- **No bouncing animations:** Use calm decelerations (`--ease-out: cubic-bezier(0.23, 1, 0.32, 1)`).
- **No pulsing green status dots:** Use purposeful status labels or the signature cycling square.
- **Case rules:** Sentence case for headlines and action labels; lowercase for **abiome**; small caps (`Alegreya SC`) for category overlines.

---

## 5. Signature Motifs

### The Cycling Square
The cycling square is the abiome punctuation mark and liveness indicator. It rotates through 90-degree steps while shifting color across the brand palette:

```css
.cycle {
  width: 0.8em;
  height: 0.8em;
  background: var(--ink);
  display: inline-block;
  flex: none;
  animation: cycle 12s var(--ease-in-out) infinite;
  animation-delay: var(--cd, 0s);
}

@keyframes cycle {
  0%, 18%   { background: var(--ink);      transform: rotate(0deg); }
  25%, 43%  { background: var(--brand);    transform: rotate(90deg); }
  50%, 68%  { background: var(--seaglass); transform: rotate(180deg); }
  75%, 93%  { background: var(--mint);     transform: rotate(270deg); }
  100%      { background: var(--ink);      transform: rotate(360deg); }
}
```

### The Bayer-Dithered Flow Field
The background runs a low-resolution WebGL Bayer 4x4 ordered dither fluid simulation with 5 discrete quantization levels, upscaled with `image-rendering: pixelated` (1 shader pixel ≈ 3 CSS pixels). It features a subtle mouse-following halo and pauses/collapses cleanly when reduced motion is preferred.

### Footer Panel Cutout
Large brand marks in footers use an SVG XOR mask on the white paper panel, cutting out the swirl shape so the living sea-glass field shows through the negative space.

---

## 6. Dense Product UI & Consoles (iso.abiome.org, AbCP, platform-main)

Dense operational consoles (code editors, multi-agent run inspectors, terminal views, file trees, sequence analyzers) require specialized handling to preserve speed and legibility while maintaining the abiome identity:

### Split of Surfaces:
1. **App Shell, Navigation, Chrome & Modals:**
   - Full abiome brand language: sea-glass field backdrop, sharp white paper mosaic panels, Alegreya serif labels and small caps overlines, brand green accents (`#006e59`), and crisp 1px borders.
2. **Dense Working Surfaces (Code Editor, Terminal, Logs, File Trees):**
   - **Legibility first:** Do not place animated shader noise behind active code editors or high-throughput log streams. The live field recedes to a static neutral tone or is omitted inside the active editor pane.
   - **Tokens maintained:** Working surfaces strictly preserve the ink (`#1d1b18`), paper (`#ffffff`), brand green (`#006e59`), and line tokens. Corners remain completely square (`border-radius: 0`).
   - **Typeface selection:** Chrome, status bars, and panel titles use `Alegreya` / `Alegreya SC`; the editor buffer and log streams use a high-legibility monospace typeface.
   - **Component usage:** Use `@abiome/ds` for shell navigation, dialogs, buttons, and settings. For internal editor/terminal canvases that already possess an integrated theme system (e.g. Monaco, CodeMirror, xterm.js), map their syntax colors to the abiome ink/paper/brand token palette rather than forcing unjustified serif typography inside code viewports.

---

## 7. The 60-Second abiome Page Recipe

When spinning up a new page or component for abiome:

1. **HTML Shell:**
   ```html
   <link rel="preconnect" href="https://fonts.googleapis.com" />
   <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
   <link href="https://fonts.googleapis.com/css2?family=Alegreya:ital,wght@0,400;0,500;0,600;0,700;1,400;1,500&family=Alegreya+SC:wght@400;500&display=swap" rel="stylesheet" />
   <canvas id="bg" aria-hidden="true"></canvas>
   <div class="grain" aria-hidden="true"></div>
   ```

2. **Base CSS:**
   ```css
   body {
     font-family: "Alegreya", "Iowan Old Style", Georgia, serif;
     color: #1d1b18;
     background: #a8d0bd;
     margin: 0;
   }
   .panel {
     background: #ffffff;
     border-radius: 0;
     border: 1px solid rgba(29, 27, 24, 0.16);
   }
   ```

3. **Interactive Link Hover:**
   ```css
   a {
     color: #4d463c;
     text-decoration: none;
     background-image: linear-gradient(#1d1b18, #1d1b18);
     background-repeat: no-repeat;
     background-size: 0% 1px;
     background-position: 0 100%;
     transition: background-size 250ms cubic-bezier(0.23, 1, 0.32, 1), color 250ms cubic-bezier(0.23, 1, 0.32, 1);
   }
   a:hover {
     color: #1d1b18;
     background-size: 100% 1px;
   }
   ```
