---
name: frontend-design
description: Use before writing or editing any front-end UI code (HTML/CSS/templates/components) in this repo. Steers away from generic, default-looking web design toward a distinctive, cohesive visual system.
---

# Frontend design

Read this before generating front-end code. The goal is to avoid the generic
"default AI web design" look (centered hero, generic blue gradient, stock
Bootstrap card grid, Inter font, emoji icons) and instead produce something
that looks intentionally designed for this product.

## Before writing code

1. Look at the existing templates/components/CSS in the repo and match the
   established visual language (colors, spacing scale, type, component
   patterns) rather than introducing a new one per page.
2. Pick a point of view: one distinctive typographic pairing, one accent
   color plus a restrained neutral palette, one spacing/radius scale. Reuse
   it everywhere instead of ad hoc values per component.
3. Prefer real icons/illustrations or none over emoji as UI iconography.

## Defaults to avoid

- Generic centered hero + subtitle + two buttons layout with no distinguishing
  detail.
- Purple/blue gradient backgrounds as a default "modern SaaS" look.
- Card grids with identical drop shadows and rounded corners on every element.
- System-default font stacks when the project already defines a type system.
- Overusing box-shadow, gradients, and rounded-everything as substitutes for
  actual layout and hierarchy decisions.

## Defaults to prefer

- Deliberate hierarchy: one clear focal element per view, generous negative
  space, alignment to a grid.
- Color used sparingly and purposefully (state, emphasis, brand) rather than
  decoratively.
- Motion/interaction only where it clarifies state change (hover, loading,
  transition) — not decorative animation.
- Accessible contrast and focus states checked, not assumed.

## Verification

After implementing, view the page/component in a browser (or via the `run`
skill if available) and check it against the surrounding pages for visual
consistency before considering the change done.
