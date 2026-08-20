---
name: web-visual-design
description: >
  Visual design guidance for building or modifying web pages and components
  with React and Tailwind CSS. Use whenever creating new pages, components,
  or layouts, or when restyling existing UI. Covers color, typography,
  spacing, borders, motion, and dark/light mode. Library-agnostic.
---

# Web Visual Design

Modern, minimal visual design for web pages built with React and Tailwind CSS.
The aesthetic reference is Vercel / shadcn-style design systems: neutral,
restrained, and content-first.

## Core philosophy

1. **UX never yields to beauty.** Every visual decision must serve usability
   first. If a choice makes the page prettier but harder to use (lower
   contrast, missing affordances, ambiguous states), it is wrong.
2. **Minimal does not mean empty.** Remove decoration, not information.
   Whitespace, hierarchy, and alignment do the design work — not gradients,
   glows, or ornament.
3. **Neutrals carry the design.** Color is a signal, not a theme. Use a
   single restrained accent and reserve strong color for meaning
   (primary action, success, warning, destructive).
4. **Consistency over cleverness.** Reuse the same spacing, radius, border,
   and type patterns everywhere. A predictable interface feels fast and
   trustworthy.

This skill is library-agnostic. Apply these rules with plain Tailwind
utilities regardless of which component library (if any) is in use.

## Color palette

Use Tailwind's `zinc` scale as the neutral backbone. It is deliberately
gray-cool and close to what Vercel and shadcn use.

| Role            | Light mode                 | Dark mode                  |
| --------------- | -------------------------- | -------------------------- |
| Page background | `white` / `zinc-50`        | `zinc-950`                 |
| Surface (card)  | `white`                    | `zinc-900`                 |
| Foreground      | `zinc-950`                 | `zinc-50`                  |
| Muted text      | `zinc-500`                 | `zinc-400`                 |
| Border          | `zinc-200`                 | `zinc-800`                 |
| Hover surface   | `zinc-100`                 | `zinc-800/50`              |
| Accent          | one color, e.g. `blue-600` | one color, e.g. `blue-400` |
| Destructive     | `red-600`                  | `red-400`                  |
| Success         | `emerald-600`              | `emerald-400`              |

Rules:

- **One accent color per product.** Pick it once, use it only for primary
  actions, active states, and focus. Never use it for large surface areas.
- **Never use pure `black`/`white` for foreground on colored surfaces** —
  use the zinc scale so text softens naturally in dark mode.
- **No decorative gradients, glassmorphism, or neon glows.** If a surface
  needs to stand out, use a border or slightly different background shade.
- **Both modes, every time.** Any class that sets a color must have its
  dark-mode counterpart (see below).

## Dark / light mode

Use Tailwind's `dark:` variant directly on utility classes, per component.
The `dark` class strategy is assumed (`.dark` on `<html>`, toggled by the
app's theme provider).

```tsx
import { type ReactNode } from "react";

function Card({ children }: { children: ReactNode }): React.JSX.Element {
  return (
    <div className="rounded-lg border border-zinc-200 bg-white p-6 dark:border-zinc-800 dark:bg-zinc-900">
      {children}
    </div>
  );
}
```

Rules:

- **Always pair.** Every color, border, background, or shadow class must be
  immediately followed by its `dark:` variant. Review diffs for unpaired
  color utilities — a light-only component is a bug.
- **Check contrast in both modes.** `zinc-500` on `white` passes AA;
  `zinc-500` on `zinc-950` does not — dark mode muted text steps up to
  `zinc-400`.
- **Do not invert colors naively.** Shadows, overlays, and accent colors need
  their own dark-mode values (usually lighter/softer), not automatic
  inversion.

## Typography

Use the system font stack (`font-sans`, Tailwind's default). No external
font dependency.

- **Headings:** `font-semibold` (600) is enough — reserve `font-bold` for
  display-scale marketing text only. Use `tracking-tight` on large headings.
- **Body:** default weight, `leading-6` or `leading-7` for paragraphs.
  Aim for ~65–75 characters per line.
- **Secondary/muted text:** `text-zinc-500 dark:text-zinc-400` and often one
  size smaller (`text-sm`).
- **Scale:** stick to Tailwind's default scale. `text-xs` for labels/captions,
  `text-sm` for dense UI, `text-base` for body, `text-lg`–`text-4xl` for
  headings. Avoid arbitrary text sizes.

## Spacing and layout

- **4px rhythm.** Use Tailwind's spacing scale only; avoid arbitrary values
  (`p-[13px]`) except for pixel-critical edge cases.
- **Generous whitespace.** When in doubt, add space. Sections breathe with
  `py-16`/`py-24`; cards and form groups use `space-y-6` or more.
- **Container:** constrain content width (e.g. `mx-auto max-w-5xl` or
  `max-w-2xl` for prose) rather than spanning the full viewport.
- **Alignment:** consistent vertical rhythm within a section — one padding
  pattern per context, repeated everywhere.

## Borders, radius, shadows

- **Subtle 1px borders over shadows.** The default separation mechanism is
  `border border-zinc-200 dark:border-zinc-800`. Shadows are not the primary
  way to delineate static content.
- **Small radius scale.** `rounded-md` for small controls (buttons, inputs),
  `rounded-lg` for containers (cards, dialogs). Never mix radii on the same
  element or nest inconsistent radii.
- **Shadows mean elevation.** Reserve shadows for elements that float above
  the page: dropdowns, dialogs, popovers — e.g.
  `shadow-lg dark:shadow-black/20`. Static cards and sections get borders,
  not shadows.

## Motion and transitions

- **Subtle and fast.** Default to `transition-colors duration-150` (or
  `duration-200`) with the default ease. Nothing animates longer than
  ~250ms except explicitly choreographed moments.
- **Micro-interactions only.** Animate state changes (hover, focus, active,
  open/close), not page load.
- **Respect reduced motion:**

```tsx
<button
  type="button"
  className="transition-colors duration-150 hover:bg-zinc-100 dark:hover:bg-zinc-800/50 motion-reduce:transition-none"
>
  Save
</button>
```

## Accessibility (non-negotiable)

- **Contrast:** WCAG AA minimum (4.5:1 body text, 3:1 large text and UI
  boundaries) in **both** modes. Verify muted text and border contrast in
  dark mode specifically.
- **Visible focus:** never remove focus outlines. Style them instead:

```tsx
<input
  type="email"
  className="rounded-md border border-zinc-200 bg-white px-3 py-2 text-sm
    focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-blue-600
    dark:border-zinc-800 dark:bg-zinc-950 dark:focus-visible:ring-blue-400"
/>
```

- **Affordances stay:** clickable elements look clickable (cursor, hover
  state, and where appropriate a visible border or background). Never hide
  affordances for visual cleanliness.
- **Semantic HTML first:** Tailwind styles semantics, it does not replace
  them. Use `<button>` for actions, `<a>` for navigation, real headings in
  order, labels tied to inputs.
- **State is communicated beyond color:** errors pair color with text and/or
  icons; active navigation states use weight or background in addition to
  hue.

## Reference components

Apply all of the above together. Note the consistent patterns: paired
`dark:` classes, border-first separation, one accent, visible focus,
system font.

### Button (primary / secondary / ghost)

```tsx
import { type ButtonHTMLAttributes } from "react";

type ButtonVariant = "primary" | "secondary" | "ghost";

interface ButtonProps extends ButtonHTMLAttributes<HTMLButtonElement> {
  variant?: ButtonVariant;
}

const variantClasses: Record<ButtonVariant, string> = {
  primary:
    "bg-blue-600 text-white hover:bg-blue-700 focus-visible:ring-blue-600 dark:bg-blue-600 dark:hover:bg-blue-500 dark:focus-visible:ring-blue-400",
  secondary:
    "border border-zinc-200 bg-white text-zinc-950 hover:bg-zinc-100 focus-visible:ring-zinc-400 dark:border-zinc-800 dark:bg-zinc-900 dark:text-zinc-50 dark:hover:bg-zinc-800 dark:focus-visible:ring-zinc-500",
  ghost:
    "text-zinc-700 hover:bg-zinc-100 focus-visible:ring-zinc-400 dark:text-zinc-300 dark:hover:bg-zinc-800/50 dark:focus-visible:ring-zinc-500",
};

function Button({
  variant = "secondary",
  className = "",
  ...props
}: ButtonProps): React.JSX.Element {
  return (
    <button
      type="button"
      className={`rounded-md px-4 py-2 text-sm font-medium transition-colors duration-150
        focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2
        focus-visible:ring-offset-white dark:focus-visible:ring-offset-zinc-950
        disabled:pointer-events-none disabled:opacity-50 motion-reduce:transition-none
        ${variantClasses[variant]} ${className}`}
      {...props}
    />
  );
}
```

### Card

```tsx
import { type ReactNode } from "react";

interface CardProps {
  title: string;
  description?: string;
  children?: ReactNode;
}

function Card({ title, description, children }: CardProps): React.JSX.Element {
  return (
    <section className="rounded-lg border border-zinc-200 bg-white p-6 dark:border-zinc-800 dark:bg-zinc-900">
      <h2 className="text-lg font-semibold tracking-tight text-zinc-950 dark:text-zinc-50">
        {title}
      </h2>
      {description !== undefined && (
        <p className="mt-1 text-sm leading-6 text-zinc-500 dark:text-zinc-400">
          {description}
        </p>
      )}
      {children !== undefined && <div className="mt-4">{children}</div>}
    </section>
  );
}
```

### Form field

```tsx
import { type InputHTMLAttributes, useId } from "react";

interface TextFieldProps extends InputHTMLAttributes<HTMLInputElement> {
  label: string;
  error?: string;
}

function TextField({ label, error, id, ...props }: TextFieldProps): React.JSX.Element {
  const generatedId = useId();
  const fieldId = id ?? generatedId;
  const errorId = `${fieldId}-error`;

  return (
    <div className="space-y-2">
      <label
        htmlFor={fieldId}
        className="block text-sm font-medium text-zinc-950 dark:text-zinc-50"
      >
        {label}
      </label>
      <input
        id={fieldId}
        aria-invalid={error !== undefined}
        aria-describedby={error !== undefined ? errorId : undefined}
        className="w-full rounded-md border border-zinc-200 bg-white px-3 py-2 text-sm
          text-zinc-950 placeholder:text-zinc-400
          focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-blue-600
          dark:border-zinc-800 dark:bg-zinc-950 dark:text-zinc-50 dark:placeholder:text-zinc-500
          dark:focus-visible:ring-blue-400"
        {...props}
      />
      {error !== undefined && (
        <p id={errorId} className="text-sm text-red-600 dark:text-red-400">
          {error}
        </p>
      )}
    </div>
  );
}
```

## Checklist

Before considering a page or component done, verify:

- [ ] Every color utility has a paired `dark:` variant.
- [ ] Text contrast passes AA in both light and dark mode.
- [ ] Focus states are visible and styled on all interactive elements.
- [ ] Exactly one accent color is in use, applied sparingly.
- [ ] Separation uses borders; shadows only on floating layers.
- [ ] Spacing uses Tailwind's scale with a consistent rhythm.
- [ ] Transitions are ≤200ms and covered by `motion-reduce:`.
- [ ] Semantic HTML elements are used for their intended purpose.
- [ ] States (error, active, disabled) are communicated beyond color alone.
