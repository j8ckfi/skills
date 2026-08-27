---
name: frontend
description: Blanket preference and craft baseline for ALL frontend engineering — new UI, components, chrome, landing pages, tools, modals, or design reviews. Enforces the CSS-cascade model of design docs (reads repo design.md / FRONTEND.md / GUIDELINES.md first), strictly isolates product brands into silos, mandates monochrome UI (system white, black, grays), bans decorative chrome, wrappers, and eyebrows, enforces Emil Kowalski motion principles (transitions.dev) with zero-animation on repeat (>10x) actions, and eliminates agent-default slop across all web and desktop surfaces. Use whenever writing, styling, or reviewing UI code across any product or stack.
license: MIT
metadata:
  version: "1.0.0"
---

# Frontend Design & Craft Standards

This document is the blanket baseline for all frontend work across every repository and stack. It is an operating procedure, not an essay.

The core principle: **UX designers have already figured it out.** Agents do not know this and default to inventing unnecessary chrome, gratuitous cards, decorative wrappers, and template aesthetics. This skill eliminates those defaults and sets an unopinionated, high-craft, functional baseline.

---

## 1. The Cascade: `design.md` as Repo Override

Design languages cascade like CSS. This skill is the **user-agent stylesheet**; local product documentation is the **author stylesheet**.

```
+-------------------------------------------------------------+
| User-Agent Baseline: skills/frontend/SKILL.md (This skill)   |
+-------------------------------------------------------------+
                              |
                              v (overridden by)
+-------------------------------------------------------------+
| Repo Author Document: design.md / FRONTEND.md / GUIDELINES  |
+-------------------------------------------------------------+
```

### Procedure:
1. **Read local design docs first:** Before creating or modifying UI, check the repo for `design.md`, `FRONTEND.md`, `GUIDELINES.md`, or a design system (`ds/`, `@theme`, `theme.css`).
2. **Local wins:** If a repo has its own design language, obey it strictly.
3. **Brand routing:** If the repository is under `abiome-org/*`, `*.abiome.org`, or uses `@abiome/ds`, stop and apply the sibling `abiome-ui` skill instead of this generic document.
4. **Persist new decisions:** If no `design.md` exists and you establish new interface decisions (radii, layout rules, component choices), write them to `design.md` at the root of the repo. Never bury design choices solely in pull request descriptions or commit messages.

---

## 2. Product Silos

Every product possesses its own sovereign design language. Never blend or import design languages across product boundaries.

- **Strict isolation:** Mari stays Mari. abiome stays abiome. A new tool gets its own intentional language.
- **No token leakage:** Never copy brand-specific palettes, custom shaders, signature motifs (such as cycling squares), or proprietary typography into a different product.
- **Reference example:** Review [Mari's `docs/FRONTEND.md`](https://github.com/j8ckfi/Mari) as an example of a product-siloed document that defines its own radii, Base UI controls, and motion boundaries without generic defaults. It is an example of product isolation, not a template to clone.

---

## 3. Functional Components, Zero Decorative Chrome

UI consists of components that perform functions and nothing more.

- **No decorative wrappers:** Do not wrap elements in redundant `<div>` containers or borders that exist only to "frame" the content.
- **No cards for the sake of cards:** If an element is a list item or a row, render a list item or a row. Do not place every single field, statistic, or paragraph inside its own rounded border card.
- **No redundant kickers/labels:** If a component already has a name, header, or obvious purpose, do not place a secondary label above it.
- **NO EYEBROWS EVER:** Strictly ban the marketing-site kicker: tiny all-caps or small-caps labels floating above headlines (e.g., `PLATFORM`, `FEATURES`, `OVERVIEW`). Ban any ornamental label or accent tag sitting above a component that is not itself the component.

---

## 4. Monochrome by Default

In the general frontend baseline, UI is strictly monochrome:

- **Palette:** System white, system black, and standard system grays (e.g., slate/neutral/zinc grays already defined in the platform).
- **No brand color:** Do not introduce arbitrary brand colors (no indigo, no violet, no teal, no amber) unless an explicit repo `design.md` dictates one.
- **System theme tracking:** Light and dark modes must follow system preferences (`prefers-color-scheme`) cleanly through CSS variables or native color schemes.
- **High contrast:** Maintain clean contrast boundaries between foreground text and surface backgrounds.

---

## 5. Motion & Responsiveness (transitions.dev)

Follow the motion principles of Emil Kowalski ([transitions.dev](https://transitions.dev/)):

- **Always responsive:** Layouts must adapt cleanly to all viewport dimensions without horizontal blowout or broken grids.
- **Custom easings:** Decelerate on entrances using `--ease-out: cubic-bezier(0.23, 1, 0.32, 1)`.
- **Short chrome durations:** All interface chrome transitions must complete in `< 300ms`.
- **Press feedback:** Active states on interactive controls should provide immediate physical feedback (e.g., `:active` scale ~0.97).
- **Origin-aware popovers:** Modals, tooltips, and menus must scale and expand from the trigger element, not from the center of the screen.
- **Prohibited motion patterns:** Never use `transition: all`, never use `ease-in` for entrances, and never use `scale(0)` entry animations.
- **Inert hidden elements:** Anything transitioned off-screen or hidden must be made non-interactive (`pointer-events-none`, `tabindex="-1"`, `aria-hidden="true"`).

### The 10x Rule (Repeat Action Invariant)
**If a user is going to do an action more than ten times, they must not see an animation for it.**
- **One-shot transitions:** Initial page load, first-time onboarding modal, or major state transitions may animate smoothly.
- **Repeat utilitarian actions:** Sending a chat message, toggling a setting switch, opening a recurring picker, typing into an input, or scrolling a list must stay still and respond instantaneously without decorative transition delays.

---

## 6. Hard Bans (Agent-Default Aesthetics)

The following defaults are banned across all frontend work:

| Banned Default | Failure Reason | Required Practice |
|---|---|---|
| **Eyebrows / kickers** | Ornamental marketing clutter that degrades interface hierarchy. | State the headline or component title directly. No floating mini-tags. |
| **Decorative card-for-a-card** | Clutters layout with unnecessary borders, paddings, and floating surfaces. | Let content sit directly on the surface or in flat structured lists. |
| **Inter / Geist / system-ui as "designed" face** | Generic template default applied without consideration. | Choose a typeface intentionally for the domain, or let system typography remain unpretentious. |
| **Indigo / violet purple primaries** | Generic shadcn/Tailwind default with zero product identity. | Strict monochrome (system white, black, neutral grays) unless repo `design.md` specifies otherwise. |
| **Mesh / aurora / hero gradients** | Visual filler that conceals weak layout and typography. | Solid surfaces, clean whitespace, and purposeful layout grids. |
| **Glassmorphism / backdrop-blur decoration** | Illegible contrast, noisy layering, and GPU waste. | Opaque surfaces with clean contrast borders. |
| **`rounded-2xl` + drop-shadow card stacks** | Floats content aimlessly and destroys alignment. | Cohesive, modest radius scale with flat 1px borders. |
| **Pulsing / pinging status dots** | Distracting fake urgency. | Static indicators, clear text labels, or purposeful bespoke loaders (e.g., progress bars, shimmers). |
| **`transition: all`** | Layout thrashing and slow composite recalculations. | Explicit transition targets: `opacity`, `transform`, `color`, `background-color`, `border-color`. |
| **Bounce / spring physics on standard chrome** | Gimmicky overshoots that slow down navigation. | Decelerating ease-out (`cubic-bezier(0.23, 1, 0.32, 1)`) under 300ms. |
| **Animating repeat interactions (>10x)** | Degrades usability on frequent interactions. | Instantaneous response for high-frequency actions (send, toggle, picker, scroll). |
| **ALL-CAPS UI labels** | Hard to scan and visually aggressive. | Sentence case for buttons, labels, headers, and tabs. |
| **Emoji as interface icons** | Inconsistent across OS platforms and misaligned baselines. | Dedicated SVG icon set (Lucide, Phosphor, Heroicons) or CSS-styled unicode symbols. |

---

## 7. Living Document

This skill is a living standard maintained to prevent recurring AI design antipatterns.

### Boundary of Responsibility:
1. **This file (`skills/frontend/SKILL.md`):** Universal baseline rules, anti-slop bans, the 10x motion rule, monochrome defaults, and the design doc cascade model.
2. **Repo design document (`design.md` / `FRONTEND.md` / `GUIDELINES.md`):** Product-specific overrides, custom radius mappings, component libraries, domain-specific controls, and product-level theme tokens.
3. **`skills/abiome-ui/SKILL.md`:** The siloed brand guidelines, WebGL flow field, and design tokens exclusive to the abiome surface.
