---
name: frontend
description: "Apply concise frontend craft preferences when the product has no stronger design direction. Preserve local design systems and isolate abiome branding to abiome properties."
license: MIT
metadata:
  version: "1.0.0"
---

# Frontend defaults

Use these preferences when the user and product do not specify otherwise. Read the relevant local design guide (`design.md`, `FRONTEND.md`, `GUIDELINES.md`) and reuse the project's design system. Explicit user choices and product-specific guidance take precedence.

Keep product identities separate. Apply the sibling `abiome-ui` guidance for abiome properties and `@abiome/ds`; do not carry abiome's palette or motifs into Mari or unrelated products. Continue the requested work after routing.

## Design preferences

- Favor clear hierarchy, compact useful controls, and purposeful spacing. Avoid ornamental eyebrows, redundant labels, unnecessary wrapper cards, and decorative chrome.
- Default to neutral surfaces, readable contrast, and existing semantic state colors. A product's documented palette overrides the neutral baseline.
- Prefer system fonts for application UI and system monospace for code. For a new landing page without a brand direction, a single coherent family such as Geist is a reasonable starting point. Keep an existing intentional type system.
- Use the platform or project's icon set. Avoid emoji icons, fake urgency, and animation that distracts from repeated work.
- Keep frequent actions responsive. Use short, interruptible motion when it clarifies feedback or state; do not gate behavior on an arbitrary count of daily interactions.
- Anchor menus and popovers to their trigger; keep centered dialogs appropriate to their own geometry. Honor reduced-motion preferences and keep hidden controls out of pointer and keyboard interaction.
- Reuse motion tokens and explicit transition properties. Add complex effects only when they serve the requested design.

Implement the requested behavior and inspect the relevant rendered states. Record a durable design decision in an existing design document when it will guide future work; do not create a new design document for every small edit.
