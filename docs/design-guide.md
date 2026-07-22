# CMMess — Design Guide (v1 starter baseline)

> **UI visual-design authority** (per `docs/authority-docs-by-area.md`). Status:
> **starter baseline, PM-authored 2026-07-22 — awaiting the Architect's design
> pass.** It exists so UI tasks build against tokens instead of raw values from
> day one; every value below is expected to be restyled, not re-plumbed. Tokens
> over raw values (architecture-facts § Styling): components consume the CSS
> custom properties in `src/index.css`, never hex literals.

## Principles (hold even after restyling)

1. **Utilitarian first.** CMMess is a shop-floor tool; density and scan-speed
   beat decoration. No animation beyond trivial transitions in v1.
2. **Status is color + text, never color alone** (color-blind safe): every
   up/down/priority/WO-status indicator pairs its color with a label.
3. **One accent, semantic colors reserved.** Red/amber/green speak only for
   status; the accent color carries interaction (buttons, links, focus).
4. **Keyboard + focus visible.** Native elements, real `<button>`/`<label>`,
   visible focus rings. No div-buttons.

## Tokens (`src/index.css` `:root` — the single source)

| Token | Value | Use |
|---|---|---|
| `--color-bg` | `#f5f6f8` | app background |
| `--color-surface` | `#ffffff` | cards, panels, table rows |
| `--color-border` | `#d4d8de` | hairlines, dividers |
| `--color-text` | `#1a2129` | primary text |
| `--color-text-muted` | `#5b6672` | secondary text, labels |
| `--color-accent` | `#1f6feb` | buttons, links, focus, active nav |
| `--color-down` / `--color-danger` | `#c93c37` | asset down, destructive, high priority |
| `--color-up` / `--color-ok` | `#1a7f37` | asset up, success |
| `--color-warn` | `#b58407` | medium priority, warnings |
| `--color-neutral-status` | `#57606a` | low priority, terminal WO states |
| `--space-1/2/3/4` | `4px / 8px / 16px / 24px` | spacing scale |
| `--radius` | `6px` | corners |
| `--font-ui` | system-ui stack | everything |
| `--font-mono` | ui-monospace stack | asset paths, ids, timestamps |
| `--text-sm/base/lg/xl` | `12px / 14px / 17px / 22px` | type scale |

## Component patterns

- **Status pill** — inline-block, `--radius`, colored left dot + label text
  (`Down 42m` / `Up`); the one shared way status renders everywhere.
- **Priority tag** — text label (`high`/`medium`/`low`) tinted with its
  semantic color; never color-only.
- **Tables** — the default list surface (WO list, history). Header row muted
  + `--text-sm`; rows on `--color-surface` with `--color-border` hairlines;
  whole row clickable when it opens a detail.
- **Forms** — stacked `<label>` above input, `--space-2` gaps; errors inline
  below the field in `--color-danger` text; submit is the single accent button.
- **Asset tree** — indented path hierarchy, `--font-mono` for the leaf path
  segment, status pill trailing each asset node.

## Layout

- **App shell:** fixed left sidebar nav (Assets · Work Orders · + New WO ·
  user/logout footer), content area right. No resizable dividers in v1 —
  if one is added, Rule 11's both-sides padding pre-flight applies.
- Content max-width none — density over centering; pad content `--space-4`.

## Out of v1 styling scope

Dark mode · theming/branding hooks · responsive/mobile layouts (Electron
desktop only) · iconography beyond text/dots.
