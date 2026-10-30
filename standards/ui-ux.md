# UI / UX standards

Derived from the prototype in this repo (`business-os/tailwind.config.js`, `business-os/src/index.css`, `business-os/src/components/ui.tsx`, `business-os/src/components/TopNav.tsx`). Reuse them; do not invent new styles.

## Layout
- [ ] **Top menu bar** with glossy mega-menus (role-filtered), Cmd/Ctrl-K command palette, language switcher. No left sidebar on desktop.
- [ ] Mobile: hamburger drawer (role-filtered groups, language), responsive grids/tables/modals, safe-area insets.
- [ ] Page = `PageHeader` + `Toolbar` + content `Card`s on the `--canvas` background.

## Theme: light glossy (dark supported)
- Font: Inter Variable, Noto Sans Devanagari, Noto Sans Telugu; base size 13 px; `tabular-nums` for numbers/money.
- Colours are CSS variables used via Tailwind `slate-*` and `brand-*` classes (they flip in `.dark`):

| Token | Light value |
|---|---|
| `brand-500` / `600` / `700` | `#6366F1` / `#4F46E5` / `#4338CA` (indigo) |
| `--grad` | `135deg #6366f1 → #8b5cf6 → #a855f7` (signature accent) |
| `--canvas` / `--surface` / `--surface-2` | `#f4f6fb` / `#ffffff` / `#f8f9fc` |
| `slate-900` (text) / `slate-500` (muted) / `slate-200` (borders) | rgb(12 18 36) / rgb(100 113 139) / rgb(228 232 240) |
| Shadows | `shadow-card`, `shadow-lift`, `shadow-glow` |

- [ ] Use only these tokens; no hex values or arbitrary Tailwind colours in components.
- [ ] Radius: `rounded-xl`/`rounded-2xl` cards, `rounded-lg` inputs/buttons, `rounded-full` badges/chips.
- [ ] Spacing on Tailwind's 4 px scale; common: `gap-2`, `gap-3`, `px-3`, `px-5`.
- [ ] Focus ring: 2 px `brand-500` outline (global `:focus-visible`); never remove it.

## Components (from `business-os/src/components/ui.tsx`)
`Button, Card, PageHeader, Kpi, Tabs, DataTable, SearchBox, Select, Segmented, Toolbar, Modal, Field/Fields, Badge, Status, Timeline, Progress, Empty, Callout`. Icons: `lucide-react`. Charts: `recharts`.
- [ ] Use these first; new shared components go into the design system package, not into a module.

## States (every screen, every data view)
- [ ] **Empty**: explain what goes here + one primary action (`Empty`).
- [ ] **Loading**: skeletons matching layout; no layout shift; spinner only for < 1 s actions.
- [ ] **Error**: plain-language message, retry, `trace_id` for support; field errors inline.
- [ ] **Offline / slow network**: `NetStatus` banner; queue safe actions; never lose typed input.
- [ ] **Partial / permission-limited / flag-off**: hide unavailable actions, keep existing data visible.

## Accessibility: WCAG 2.2 AA
- [ ] Contrast ≥ 4.5:1 text, 3:1 UI; tap targets ≥ 44 px on mobile (24 px min desktop).
- [ ] Every input has a label; errors linked via `aria-describedby`; landmarks and headings in order.
- [ ] Full keyboard operation, visible focus, no keyboard traps; respects `prefers-reduced-motion`.
- [ ] axe-core clean in E2E.

## i18n: EN / HI / TE
- [ ] No hard-coded user-facing strings; use `business-os/src/lib/i18n.ts` keys. All three languages ship together.
- [ ] Format numbers/money with `en-IN` grouping (₹1,23,456.00) and dates per locale; allow text expansion (+40%).

## MSME-friendly
- [ ] Plain words, no jargon; the AI proposes, the user approves in one tap.
- [ ] Smart defaults (last-used values, GST from HSN, customer from GSTIN lookup); minimum required fields.
- [ ] **Keyboard-first entry**: Tab order follows the paper form, Enter to add line, Ctrl/Cmd-S save, Esc close, Cmd-K anything.
- [ ] Mobile-responsive: works at 360, 375, 390, 768 px with no horizontal overflow.
