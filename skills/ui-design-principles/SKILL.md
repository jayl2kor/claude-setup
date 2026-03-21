---
name: ui-design-principles
description: Activate when making decisions about visual design, layout, typography, color, spacing, iconography, or dark mode in a UI. Trigger when the user asks about design systems, visual hierarchy, contrast ratios, color palettes, font pairing, spacing scales, grid systems, or when reviewing a design for visual correctness and platform conformance (Material Design 3, Apple HIG).
version: 1.0.0
---

# UI Design Principles

Expert-level visual design guidance grounded in Material Design 3, Apple Human Interface Guidelines, Gestalt psychology, and WCAG standards. Principles are platform-agnostic unless noted.

## When to Activate

- Choosing or critiquing a color palette, contrast ratio, or semantic color role
- Designing or auditing a type scale, font pairing, or line-height rule
- Establishing a spacing system or grid for a new design or component library
- Applying visual hierarchy to a layout (what draws the eye first, second, third)
- Making iconography decisions (style, size, labeling)
- Designing or reviewing dark mode variants
- Evaluating a design against Material Design 3 or Apple HIG principles

---

## Platform Design Languages

### Material Design 3 (Google)

MD3's three core principles and their design implications:

**1. Personal** — M3 introduces dynamic color (Material You): the system extracts a five-tone palette from the user's wallpaper and maps it to semantic color roles (`primary`, `secondary`, `tertiary`, `error`, `surface`, `outline`). Never hard-code hex values for semantic colors; always reference the role. This ensures the palette works in both light and dark schemes automatically.

**2. Adaptive** — Layouts must respond across the canonical window size classes: Compact (< 600 dp), Medium (600–839 dp), Expanded (≥ 840 dp). Navigation pattern changes with size: bottom navigation bar → navigation rail → navigation drawer. Content panels go from single-pane to two-pane at Medium/Expanded.

**3. Expressive** — Motion, shape (rounded corners via ShapeKeySystem: Extra Small 4 dp → Extra Large 28 dp), and elevation (surface tones, not drop shadows) create personality without sacrificing function. Elevation is expressed as a tonal overlay (`surface + primary at N% opacity`), not a shadow color.

### Apple Human Interface Guidelines

Six foundational principles and their actionable constraints:

**Aesthetic Integrity** — Visual design must support, not decorate. Every embellishment must serve comprehension or delight; remove what does neither.

**Consistency** — Use platform-native controls (UIButton, SwiftUI Button) before building custom ones. Custom controls require matching the platform's interaction model exactly (tap targets, gestures, animations).

**Direct Manipulation** — Prefer direct gesture interaction over menus. Drag, pinch, swipe. When a gesture triggers an action, provide immediate visual confirmation.

**Feedback** — Every action receives feedback within 100 ms. Beyond 100 ms: show a spinner. Beyond 1 s: show progress. Beyond 10 s: let the user cancel.

**Metaphors** — Familiar real-world metaphors (folder, trash, swipe to delete) lower learning cost. Do not invent novel interaction models when a platform convention already exists.

**User Control** — Destructive actions (delete, overwrite, send) must require confirmation. Never auto-trigger irreversible actions. Always provide undo when technically feasible.

---

## Visual Hierarchy and Gestalt Principles

Visual hierarchy tells the eye where to look, in what order. It is built from contrast differences in size, weight, color, and position.

### Gestalt Laws Applied to Layout

**Proximity** — Elements close together are perceived as related. Use spacing deliberately: items in the same group share tight spacing (4–8 px gap); items in different groups have larger separation (24–48 px). Never use the same gap between unrelated and related items.

```
❌ BAD — equal spacing makes grouping ambiguous
[Label]  [Input]  [Label]  [Input]

✅ GOOD — tight intra-group gap, larger inter-group gap
[Label]
[Input]          ← 8 px gap between label and input
                 ← 24 px gap between fields
[Label]
[Input]
```

**Similarity** — Elements that share color, shape, or size appear related. Use it to signal interaction affordance (all blue text = tappable links). Violate it deliberately only to signal uniqueness (a primary CTA that differs from secondary buttons).

**Continuity** — The eye follows implied lines. Align elements on a grid so the eye moves predictably through the layout. Misaligned elements create visual noise and imply unintended relationships.

**Closure** — The eye completes incomplete shapes. Use it for loading skeletons (rectangles imply content), progress rings, and overlapping cards. A card that bleeds off the edge implies more content is horizontally scrollable — use this intentionally, not accidentally.

**Figure-Ground** — The subject must contrast against its background. Ensure the focal element has enough tonal contrast to separate from the surface. Low-contrast text on a patterned background is a figure-ground failure.

### Hierarchy in Practice: Three-Level Rule

Limit visual emphasis to three levels per screen. More levels create visual noise.

```
Level 1 — Page title or primary action (largest, highest contrast, heaviest weight)
Level 2 — Section headers or secondary content (medium size/weight)
Level 3 — Body copy, metadata, labels (smallest, lowest contrast, lightest weight)
```

If a design requires a fourth level, promote one level or consolidate content — do not add a fourth tier.

---

## Typography

### Type Scale

Use a modular type scale. The MD3 type scale and Apple's Dynamic Type both define named roles — use them, do not invent arbitrary sizes.

| Role (MD3 name) | Size | Weight | Line height | Use for |
|---|---|---|---|---|
| Display Large | 57 sp/pt | Regular 400 | 64 | Hero headers, marketing |
| Headline Medium | 28 sp/pt | Regular 400 | 36 | Page titles |
| Title Large | 22 sp/pt | Regular 400 | 28 | Card titles, dialog headers |
| Body Large | 16 sp/pt | Regular 400 | 24 | Primary body copy |
| Body Medium | 14 sp/pt | Regular 400 | 20 | Secondary body, captions |
| Label Large | 14 sp/pt | Medium 500 | 20 | Button labels, tabs |
| Label Small | 11 sp/pt | Medium 500 | 16 | Badges, chips, timestamps |

Never size body copy below 14 sp/pt for primary reading; below 12 sp/pt for any content that must be read (not decorative).

### Line Height

Line height (leading) must be proportional to font size. Use a ratio of 1.4–1.6× for body copy. Tighter ratios for large display text.

```
Body (16 px) → line-height: 24 px  (1.5× ratio)    ✅
Body (16 px) → line-height: 16 px  (1.0× ratio)    ❌ Lines collide, illegible
Headline (32 px) → line-height: 40 px (1.25× ratio) ✅
```

### Measure (Line Length)

Optimal reading measure: 45–75 characters per line for body copy. Enforce with `max-width: 65ch` on prose containers.

```css
/* ✅ GOOD — prose is constrained */
.article-body { max-width: 65ch; }

/* ❌ BAD — full-width prose on large screens forces long saccades */
.article-body { width: 100%; }
```

### Font Pairing

Pair a display/heading typeface with a reading typeface. Rules:
- One serif + one sans-serif is reliable
- Two sans-serifs must differ strongly in weight or width (not two Regular 400 fonts of similar x-height)
- Never use more than two typefaces in a single product (not counting a monospace font for code)

```
✅ GOOD pairings:
  Playfair Display (serif) + Inter (sans-serif)
  DM Sans (geometric) + DM Mono (monospace for code)

❌ BAD pairings:
  Roboto + Open Sans  (too similar — no contrast)
  Three different sans-serifs  (unnecessary complexity)
```

---

## Color Theory for UI

### Color Harmony Systems

| Harmony | Construction | When to use |
|---|---|---|
| Complementary | Opposite on wheel (e.g., blue + orange) | High contrast CTAs, emphasis |
| Analogous | Adjacent colors (e.g., blue, blue-violet, violet) | Calm, cohesive palettes |
| Triadic | Three equidistant (e.g., red, yellow, blue) | Vibrant; use with restraint |
| Split-Complementary | Base + two adjacent to its complement | Balanced contrast with variety |

For product UI, analogous palettes with a single complementary accent are most reliable.

### Contrast Ratios — WCAG

WCAG defines contrast as the ratio of relative luminance between foreground and background.

| Level | Ratio | Applies to |
|---|---|---|
| AA — Normal text | 4.5 : 1 | Text < 18 pt (or < 14 pt bold) |
| AA — Large text | 3.0 : 1 | Text ≥ 18 pt (or ≥ 14 pt bold) |
| AA — UI components | 3.0 : 1 | Input borders, icons, focus indicators |
| AAA — Normal text | 7.0 : 1 | Highest legibility requirement |
| AAA — Large text | 4.5 : 1 | Large text at highest legibility |

```
✅ #FFFFFF on #767676  → 4.54 : 1  (passes AA normal text)
❌ #FFFFFF on #949494  → 2.85 : 1  (fails AA for normal text)
✅ #000000 on #FFFFFF  → 21.0 : 1  (passes AAA)
```

Never rely on color alone to convey meaning (status, error, success). Always pair color with a text label, icon, or pattern.

### Semantic Color Roles

Define semantic roles, not raw colors. This makes theming and dark mode feasible.

```
Primary       — brand actions, interactive controls
On-Primary    — content that sits on top of Primary
Secondary     — complementary actions, chips, filters
Surface       — card and sheet backgrounds
On-Surface    — text and icons on Surface
Surface Variant — subtle container differentiation
Outline       — border and divider color
Error         — destructive actions, validation failure
```

Map each role to a light-scheme hex and a dark-scheme hex. Never hard-code a specific hex in component code.

---

## Spacing Systems

### 8-Point Grid

All spacing values must be multiples of 8 px. For tight spacings inside components, use multiples of 4 px.

```
4 px  — intra-component tight spacing (icon to label, badge offset)
8 px  — default component internal padding (small chips, compact rows)
16 px — standard padding (cards, list items, input fields)
24 px — section separation within a screen area
32 px — major section breaks, dialog padding
48 px — page-level vertical rhythm between sections
64 px — hero/display sections, large visual breathing room
```

```
❌ BAD — arbitrary spacings
padding: 13px 17px;
margin-bottom: 22px;

✅ GOOD — grid-aligned spacings
padding: 12px 16px;   /* 12 is 3×4; 16 is 2×8 */
margin-bottom: 24px;  /* 3×8 */
```

### Touch Target Sizing

Minimum interactive target: 44×44 pt (Apple HIG) / 48×48 dp (Material Design). Visually smaller elements (16 dp icons, small checkboxes) must have invisible extended touch areas.

```css
/* ✅ Extended touch target on small icon button */
.icon-button {
  width: 24px;
  height: 24px;
  padding: 12px;  /* total touch area: 48px × 48px */
}
```

---

## Iconography

### Style Consistency

All icons in a system must share the same visual style: stroke weight, corner radius, size grid, fill/outline convention. Mixing stroke icons with filled icons signals lack of system thinking.

| Attribute | MD3 Material Symbols | Apple SF Symbols | Recommendation |
|---|---|---|---|
| Size grid | 24 dp optical | 24 pt optical | Use system icons at native sizes |
| Weights | 100–700 (variable) | Ultralight–Black | Match icon weight to surrounding text weight |
| Fill | Fill 0/1 toggle | Regular/Fill | Use fill for selected state, outline for default |
| Corner radius | Rounded preferred | Automatic | Match product shape language |

### Icon Labeling

Icons without text labels must have accessible names (`aria-label`, `title`, platform equivalent). Icons paired with a text label are decorative and must be hidden from assistive technology (`aria-hidden="true"`).

Use text labels alongside icons whenever the icon's meaning is not universally recognized. A floppy disk for "save" requires a label for new users. A standard X for "close" is widely understood but still benefits from a label.

---

## Dark Mode Design

Dark mode is not an inverted light mode. It requires purpose-built decisions on every design token.

### Elevation and Layering

In light mode, elevation uses drop shadows. In dark mode, shadows are invisible against dark backgrounds. Use tonal elevation: higher surfaces get a higher-opacity overlay of the primary color.

```
MD3 dark mode surface elevations:
  Level 0 (base surface): #1C1B1F          (background color)
  Level 1 (+5% primary overlay):  #25232A  (cards, sheets)
  Level 2 (+8% primary overlay):  #2B2930  (dropdowns, menus)
  Level 3 (+11% primary overlay): #302E35  (dialogs)
  Level 4 (+12% primary overlay): #312F37  (navigation bar)
  Level 5 (+14% primary overlay): #34323A  (modal scrim above)
```

### Color Adjustments in Dark Mode

- Saturated colors must be desaturated and lightened for dark backgrounds. A vivid red (#D32F2F) that works on white becomes #EF9A9A on dark (pastel, not vivid).
- Do not use pure black (#000000) backgrounds — use dark grey (#121212 or #1C1B1F). Pure black creates excessive contrast against content and looks cheap.
- Do not use pure white (#FFFFFF) body text on dark — use #E6E1E5 or similar off-white to reduce eye strain.
- Reduce image brightness and saturation by 10–15% in dark mode so vivid images do not cause glare.

### Common Dark Mode Mistakes

```
❌ Inverting all colors (white→black, black→white) — breaks semantic roles
❌ Using the same shadow as light mode — invisible on dark surfaces
❌ Pure black (#000000) app background — harsh, undesirable contrast
❌ Same brand red/blue saturation — saturated hues are harsh on dark backgrounds
❌ Forgetting to test: screenshots, maps, charts, third-party embeds
```

### Implementation Pattern

Use CSS custom properties (or design tokens) so dark mode is a single token swap, not scattered overrides.

```css
:root {
  --color-surface: #FFFBFE;
  --color-on-surface: #1C1B1F;
  --color-primary: #6750A4;
}

@media (prefers-color-scheme: dark) {
  :root {
    --color-surface: #1C1B1F;
    --color-on-surface: #E6E1E5;
    --color-primary: #D0BCFF;
  }
}

/* ✅ Components reference tokens, never hard-coded colors */
.card {
  background-color: var(--color-surface);
  color: var(--color-on-surface);
}
```

---

## Decision Criteria Summary

| Decision | Primary signal | Fallback |
|---|---|---|
| Font size below 16 px body? | Fail — increase to 14 px minimum; 16 px preferred | Accept 14 px only for secondary/metadata text |
| Contrast below 4.5:1 for body text? | Fail — adjust foreground or background | Target ≥ 7:1 for AAA where feasible |
| Spacing not on 4/8 px grid? | Reject — align to nearest grid value | 4 px increments for sub-component spacing |
| Touch target below 44 pt / 48 dp? | Fail — add padding to extend hit area | Never reduce below 44×44 |
| Icon without label in ambiguous context? | Add visible text label | Add aria-label at minimum |
| Dark mode: using light-mode color directly? | Reject — create dedicated dark-mode tokens | Check every color role independently |
| More than two typefaces? | Reject — remove one | Monospace for code is exempt from the limit |
