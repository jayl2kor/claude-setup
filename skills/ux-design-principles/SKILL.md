---
name: ux-design-principles
description: Activate when designing user flows, evaluating usability, structuring forms, planning information architecture, applying accessibility standards, or making decisions about interaction patterns, feedback loops, progressive disclosure, empty states, or error states. Trigger when the user asks about WCAG compliance, Nielsen heuristics, cognitive load, Fitts's Law, mobile-first design, navigation patterns, or card sorting.
version: 1.0.0
---

# UX Design Principles

Expert-level user experience guidance grounded in established usability research, WCAG 2.2, cognitive psychology, and proven interaction design patterns.

## When to Activate

- Evaluating a design or flow against Nielsen's 10 usability heuristics
- Reducing cognitive load in a complex workflow or form
- Designing or auditing user flows (happy path, error states, empty states)
- Implementing WCAG 2.1/2.2 Level AA accessibility requirements
- Structuring form fields, validation, and error messaging
- Applying mobile-first or responsive design constraints
- Planning information architecture, navigation hierarchies, or card-sorting activities
- Choosing between interaction patterns (modals, inline editing, progressive disclosure)

---

## Nielsen's 10 Usability Heuristics

These are diagnostic criteria, not guidelines to apply blindly. Use them to identify specific violations, then apply targeted fixes.

**1. Visibility of System Status**
The system must always communicate what is happening, within a reasonable time. Thresholds: feedback within 100 ms feels instantaneous; 100 ms–1 s requires no indicator; 1–10 s requires a spinner or progress bar; > 10 s requires a progress indicator with estimated time and a cancel option.

```
❌ BAD — button does nothing visible after click for 2 seconds
✅ GOOD — button enters loading state immediately (100 ms), spinner appears, label changes to "Saving…"
```

**2. Match Between System and the Real World**
Use words, icons, and concepts the user already knows. Avoid internal jargon, error codes, and engineering terms in user-facing copy. "Error 403" → "You don't have permission to view this page." "Null result" → "No results found."

**3. User Control and Freedom**
Every action must be undoable or escapable. Support Ctrl+Z, swipe-to-dismiss, cancel buttons on dialogs, and confirmation steps before irreversible actions. If an action cannot be undone, say so explicitly in the confirmation dialog.

**4. Consistency and Standards**
Follow platform conventions. Users should not have to wonder whether different words, situations, or actions mean the same thing. Save must always be Save (not Submit, Confirm, or Apply interchangeably).

**5. Error Prevention**
Prevent errors before they occur. Disable submit until required fields are complete. Show format hints inline (e.g., "MM/DD/YYYY" placeholder). Confirm destructive actions with a dialog that describes the irreversible consequence.

**6. Recognition Over Recall**
Minimize the user's memory load. Show options rather than requiring recall. Search autocomplete, recently used items, inline help text, and context-sensitive suggestions all reduce recall demand. Never ask the user to remember information from one screen to enter it on another.

**7. Flexibility and Efficiency of Use**
Support both novice and expert paths. Provide keyboard shortcuts for power users while keeping discoverable UI for newcomers. Do not hide advanced features so well that experienced users cannot find them.

**8. Aesthetic and Minimalist Design**
Every piece of information on screen competes for attention. Remove information that does not help the user complete their current task. "Perfection is achieved not when there is nothing more to add, but when there is nothing left to take away." — apply ruthlessly.

**9. Help Users Recognize, Diagnose, and Recover from Errors**
Error messages must: (a) be in plain language, (b) precisely describe the problem, (c) suggest a constructive solution. Never show only an error code. Never show a generic "Something went wrong."

```
❌ BAD  — "Error: 422 Unprocessable Entity"
❌ BAD  — "Something went wrong. Please try again."
✅ GOOD — "Your email address already has an account. Sign in instead, or reset your password."
```

**10. Help and Documentation**
Help should be unnecessary for well-designed flows. When it is needed, it must be searchable, task-oriented, and specific (not a wall of text). Contextual help (inline tooltip, progressive disclosure) outperforms a separate Help Center for most cases.

---

## Cognitive Load Theory

Reduce the mental effort required at every step. Three types of cognitive load:

**Intrinsic** — inherent complexity of the task (cannot be eliminated, only scaffolded)
**Extraneous** — complexity introduced by poor design (eliminate this entirely)
**Germane** — load that builds mental models (invest in this through good onboarding)

### Miller's Law — 7 ± 2

Working memory holds approximately 7 (± 2) chunks of information. For UI, this means:
- Navigation menus: limit to 5–7 top-level items
- Step indicators: break workflows longer than 7 steps into phases with their own sub-steps
- Dropdowns: group options when the list exceeds 7 items; use search for lists > 10

```
❌ BAD — main navigation with 11 items
✅ GOOD — 5 primary nav items; secondary/less-used items behind a "More" menu
```

### Hick's Law — Decision Time Grows Logarithmically

Decision time increases with the number and complexity of choices. The more options, the longer it takes to decide.

- Reduce the number of primary choices on any screen
- Pre-select the most common option (safe default)
- Use progressive disclosure to hide advanced options until needed
- A wizard (sequential steps) outperforms a single form with 20 fields for complex tasks

```
❌ BAD — 12 filter options visible simultaneously on a results page
✅ GOOD — 3 primary filters visible; "More filters" reveals advanced options
```

### Fitts's Law — Acquisition Time Depends on Target Size and Distance

`T = a + b × log₂(D/W + 1)` — time to reach a target grows with distance and shrinks with target width.

Practical rules:
- Primary CTAs must be large (minimum 44×44 pt / 48×48 dp) and positioned where the thumb or cursor naturally rests
- Destructive actions must be small and far from the primary action (prevent accidental taps)
- Place the most-used actions nearest to the starting cursor/thumb position
- On mobile, bottom-of-screen placement for primary actions (thumb zone); top-right for secondary actions

```
❌ BAD — "Delete" button next to "Save", same size
✅ GOOD — "Save" large, bottom-right; "Delete" small, top or secondary location with confirmation dialog
```

---

## WCAG 2.1 / 2.2 Level AA Requirements

Level AA is the legal minimum in most jurisdictions (ADA, EN 301 549, AODA). Design for AA by default; target AAA for reading-intensive content.

### The Four Principles (POUR)

| Principle | Core requirement |
|---|---|
| **Perceivable** | All content can be perceived by all senses (not just sight) |
| **Operable** | All functionality available via keyboard; no seizure-inducing content |
| **Understandable** | Language is clear; UI behaves predictably |
| **Robust** | Content is parseable by current and future assistive technologies |

### Key AA Success Criteria with Specific Values

**1.1.1 Non-text Content** — Every image has alt text. Decorative images use `alt=""`. Complex images (charts, diagrams) have long descriptions.

**1.3.1 Info and Relationships** — Structure conveyed visually must be programmatically determinable. If a heading looks like a heading, it must be `<h1>`–`<h6>`, not a styled `<div>`.

**1.4.1 Use of Color** — Do not use color as the only visual means of conveying information. Always pair color with a text label, icon, pattern, or shape.

**1.4.3 Contrast (Minimum)** — Normal text: 4.5:1. Large text (≥ 18 pt or ≥ 14 pt bold): 3:1. Applies to text rendered on any background.

**1.4.4 Resize Text** — Text must be resizable up to 200% without loss of content or functionality. No fixed-pixel font sizes that resist browser zoom.

**1.4.10 Reflow (2.1)** — Content must reflow to a single column at 320 px wide (equivalent to 400% zoom on a 1280 px viewport) without horizontal scrolling or loss of content.

**1.4.11 Non-text Contrast (2.1)** — UI components (input borders, button boundaries, checkbox outlines) and graphical objects must have 3:1 contrast against adjacent colors.

**1.4.12 Text Spacing (2.1)** — No content loss when: line height ≥ 1.5× font size; paragraph spacing ≥ 2× font size; letter spacing ≥ 0.12× font size; word spacing ≥ 0.16× font size.

**2.1.1 Keyboard** — All functionality available via keyboard. No keyboard traps (focus must never be locked in a component from which the user cannot escape with Tab/Escape).

**2.4.3 Focus Order** — Focus order matches reading/logical order. Never use `tabindex` values above 0 to reorder focus; fix the DOM order instead.

**2.4.7 Focus Visible** — Every focusable element has a visible focus indicator. The default browser outline must never be removed without a superior replacement.

**2.4.11 Focus Appearance (2.2)** — Focus indicator must have ≥ 3:1 contrast against adjacent colors AND enclose the component with a perimeter ≥ 2 CSS pixels.

**3.3.1 Error Identification** — If an input error is detected, the item in error is identified and the error is described to the user in text.

**3.3.2 Labels or Instructions** — Labels or instructions are provided when content requires user input.

**4.1.2 Name, Role, Value** — All UI components have an accessible name, role, and state determinable programmatically. Custom controls must use appropriate ARIA.

---

## Interaction Design Patterns

### Progressive Disclosure

Show only what the user needs for their current task. Reveal complexity on demand.

```
✅ Use progressive disclosure when:
  - A form has > 5 fields and < 30% of users need the advanced fields
  - A settings page has beginner and expert settings
  - A results list has summary and detail views

❌ Do not use it when:
  - The hidden content is needed by most users (hiding it adds friction)
  - The disclosure pattern is unfamiliar to the audience (adds cognitive load)
```

### Affordances

An affordance is a visual or behavioral cue that signals how an element is used.

- Buttons must look pressable (sufficient contrast, visible boundary or fill, label)
- Links must look tappable (underline or distinct color + underline on hover)
- Draggable items must signal draggability (grip handle icon, cursor: grab)
- Input fields must look editable (border, background color different from surface, placeholder)

Removing affordances to achieve visual minimalism is a usability tradeoff, not a free simplification.

### Feedback Loops

Every user action must produce feedback. Feedback types by urgency:

| Trigger | Feedback type | Timing |
|---|---|---|
| Hover / focus | Visual state change | Immediate (< 16 ms, 60 fps) |
| Button tap / click | Loading state on element | < 100 ms |
| Form submit | Loading state, then success/error | < 100 ms for state change |
| Long operation | Progress indicator + estimated time | On start of operation |
| Background sync | Status in notification area | On completion |
| Destructive action | Confirmation dialog before execution | Before action, not after |

---

## User Flow Design

### Happy Path vs. Error States

Design the error states before you finish the happy path. Error states are where most UX debt accumulates.

For every user flow, identify:

1. **Happy path** — all inputs valid, all services available, user has permissions
2. **Validation errors** — field-level and form-level error states
3. **Permission errors** — user lacks access; explain why and provide a path forward
4. **Network/service errors** — distinguish between retry-able (503) and non-retry-able (404) errors
5. **Timeout states** — what happens if an operation exceeds the user's expectation window
6. **Partial success** — some items succeeded, some failed; show granular status

### Empty States

Empty states are high-leverage moments. They appear when a user is new, has cleared content, or a search returns nothing.

```
Every empty state needs three elements:
  1. Explanation — what is empty and why
  2. Benefit — what they will get by filling it
  3. Action — a direct CTA to fill it

❌ BAD empty state: "No results."
✅ GOOD empty state:
  Illustration or icon representing the content type
  "You haven't added any products yet."
  "Add your first product to start selling."
  [Add product] button
```

For search empty states, offer alternatives: "No results for 'red shoes'. Try 'shoes' or browse all categories."

---

## Form Design

### Field Ordering and Grouping

- Order fields by logical sequence, not database column order
- Group related fields under a visible label (fieldset/legend or visual section header)
- Place primary identifying fields (name, email) before secondary fields
- Place optional fields after required ones; mark optionals with "(optional)", not required with "*" alone

### Labels

Labels must be persistent and visible — never use placeholder text as the only label. Placeholder text disappears on focus and has insufficient contrast (WCAG 1.4.3 failure in most implementations).

```
❌ BAD — placeholder as label
  [Enter your email address          ]

✅ GOOD — persistent label above field
  Email address
  [                                  ]
  Placeholder hint optional: "you@example.com"
```

### Inline Validation Timing

| Validation moment | When to use |
|---|---|
| On blur (field loses focus) | Default; most fields |
| On change (real-time) | Format-sensitive fields (credit card, phone); only after first blur |
| On submit only | Simple short forms (2–3 fields); avoids premature errors |

Never show an error before the user has had a chance to complete the field (do not validate on keydown for empty field checks).

### Error Message Placement and Content

- Place errors immediately adjacent to the failed field (below the input, above helper text)
- Use `role="alert"` and `aria-describedby` to associate the error with the field programmatically
- Error messages must specify what is wrong and how to fix it

```
❌ "Invalid input"
❌ "Error"
✅ "Enter a valid email address (e.g., you@example.com)"
✅ "Password must be at least 8 characters and include one number"
```

---

## Mobile-First and Responsive Design

### Mobile-First Principle

Design and build the mobile (smallest) layout first. Add complexity for larger screens. This forces prioritization — only essential content survives the small-screen constraint.

```css
/* ✅ Mobile-first: base styles for small screens, enhance upward */
.card-grid {
  display: grid;
  grid-template-columns: 1fr;           /* single column on mobile */
  gap: 16px;
}

@media (min-width: 600px) {
  .card-grid {
    grid-template-columns: repeat(2, 1fr);  /* 2-col on tablet */
  }
}

@media (min-width: 960px) {
  .card-grid {
    grid-template-columns: repeat(3, 1fr);  /* 3-col on desktop */
  }
}

/* ❌ Desktop-first: starts complex and overrides down — harder to maintain */
.card-grid {
  grid-template-columns: repeat(3, 1fr);
}
@media (max-width: 960px) { ... }
@media (max-width: 600px) { ... }
```

### Breakpoint Selection

Do not choose breakpoints by device (iPhone, iPad, MacBook). Choose breakpoints where the content breaks — where the layout becomes uncomfortable or the line length exceeds the readable measure.

Common canonical breakpoints aligned to MD3 window size classes:

| Name | Range | Navigation pattern |
|---|---|---|
| Compact | 0–599 px | Bottom navigation bar |
| Medium | 600–839 px | Navigation rail |
| Expanded | 840–1199 px | Navigation drawer (collapsible) |
| Large | 1200–1599 px | Navigation drawer (persistent) |
| Extra-large | ≥ 1600 px | Navigation drawer + content max-width |

### Thumb Zone

On mobile, design primary interactions for the natural thumb reach zone (bottom 60% of screen, centered). Place:
- Primary CTAs: bottom of screen (floating action button, bottom bar)
- Destructive or rarely used actions: top of screen (harder to reach accidentally)
- Navigation: bottom bar for 3–5 destinations; hamburger only if navigation is deeply secondary

---

## Information Architecture

### Card Sorting

Card sorting is the primary research method for validating navigation labels and groupings before building. Two types:

**Open card sort** — participants group content without predefined categories. Use to discover natural mental models for the first draft of navigation.

**Closed card sort** — participants assign content to predefined categories. Use to validate a proposed navigation structure before implementation.

Minimum sample size: 15–20 participants for open sorts (saturation typically reached by 15). Agreement threshold: if < 50% of participants agree on a grouping, the category label or scope is ambiguous — revise and re-test.

### Navigation Patterns

| Pattern | When to use | When not to use |
|---|---|---|
| Top navigation bar | 3–7 primary destinations; content-forward | Mobile (no thumb reach) |
| Bottom navigation bar | 3–5 primary destinations on mobile | > 5 items (causes crowding) |
| Navigation drawer | 5+ destinations; deep hierarchy | Simple apps (overkill) |
| Tabs | Sub-navigation within a section | Top-level navigation |
| Breadcrumbs | Deep hierarchies (3+ levels); e-commerce, docs | Flat single-level sites |
| Mega-menu | 20+ navigation items with subcategories | Mobile (use drawer instead) |

### Content Hierarchy Rules

- Prioritize by frequency of use: the 20% of content used 80% of the time must be at the top level
- Maximum navigation depth: 3 levels for most products; 4 levels only for very large content catalogs
- Cross-linking replaces deep hierarchies: a product can live in one place in the taxonomy but be reachable from multiple contextual paths

---

## Common UX Anti-Patterns to Avoid

**Dark patterns** — Deceptive UX that tricks users into unintended actions. Examples to never implement:
- Hidden costs (adding fees at checkout not shown in cart)
- Roach motel (easy to subscribe, hard to cancel)
- Confirmshaming ("No thanks, I don't want to save money")
- Misdirection (prominent "Accept all" button, hard-to-find "Reject all")

**Premature optimization of the happy path** — Designing the happy path to be frictionless while leaving error and edge-case states broken or missing. Broken error states erode trust more than slightly slower happy paths.

**Modal overuse** — Use modals only for: (a) content requiring user decision before proceeding, (b) content that would lose context if navigated away. Do not use modals for notifications, success states, or information that does not require user action.

**Infinite scroll without pagination fallback** — Infinite scroll makes it impossible to bookmark a position, share a specific page, or use keyboard navigation efficiently. For content that must be shareable or navigable, paginate.

**Disabling the browser back button** — Single-page apps must update the browser history stack on navigation. The back button is the most-used navigation control on mobile; breaking it breaks user trust.

**Form autofill suppression** — Never use `autocomplete="off"` on login, address, or payment forms. Autofill is an accessibility aid for users with motor impairments and a usability aid for all users.
