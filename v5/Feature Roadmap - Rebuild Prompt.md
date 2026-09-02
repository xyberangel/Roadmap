# Recreation Prompt: Feature Roadmap Tool

Build a **single-page, self-contained interactive Feature Roadmap timeline web app**. It imports product roadmap data from Excel/CSV, renders it as a draggable Gantt-style swimlane timeline across six quarters (zoomable down to months or sprints), and exports back to Excel or as an image. All data lives in the browser (localStorage) — nothing is uploaded.

## Tech & dependencies
- Single HTML page. Font: **Arial** (system font, no external font loading) throughout, including quarter/sprint labels.
- Two CDN libraries:
  - **SheetJS / xlsx** (v0.18.5) — parsing & writing `.xlsx`/`.csv`.
  - **html2canvas** (v1.4.1) — JPEG export of the board.
- Page background `#EEF0F4`. App is a centered white card, `max-width:1600px`, `border-radius:14px`, `1px solid #DEE2EA` border, soft shadow `0 12px 34px rgba(20,30,60,0.07)`.
- Supports a light/dark theme via CSS custom properties on a `.app-root[data-theme]` wrapper (see Settings below) — every surface, border, and text color used throughout must reference the tokens, not hardcoded hex, so dark mode reskins correctly.

## Color system (light theme tokens)
- `--page-bg:#EEF0F4; --surface:#FFFFFF; --surface-2:#F8F9FB; --border:#DEE2EA; --border-soft:#E7EAF1; --divider:#EEF0F4; --text:#1C2230; --text-2:#5A6478; --text-3:#9AA3B5; --text-faint:#8A93A6; --chip-bg:#ECEEF3; --ink:#1B2A52; --shadow:0 12px 34px rgba(20,30,60,0.07)`.
- Dark theme swaps these to a navy/charcoal surface set (`--page-bg:#0E1117`, `--surface:#161B24`, `--text:#E7EAF0`, etc.) — same variable names, different values.
- Status message colors: ok `#138A72`, error `#C0392B`, neutral `#9AA3B5`.
- Navy brand ink used for headers/modals/primary buttons: `#1B2A52`.
- **Swimlane palette** — pastel bar fill + darker "ink" text color + soft background, assigned by matching the swimlane name against regex presets first, then falling back to a rotating palette:
  - Capex/Capital → fill `#AFCBF2`, ink `#2C508A`, soft `#EDF3FC`, caption "Capital expenditure · New features"
  - Tech debt/infra → fill `#ECD49C`, ink `#8A5E18`, soft `#FBF3E2`, caption "Technical debt · Infrastructure"
  - Support/maintenance/bug → fill `#CABFEC`, ink `#4C3D89`, soft `#EFEBF9`, caption "Maintenance · Bug fixes"
  - Modernisation → fill `#EBBECB`, ink `#97435C`, soft `#FBEDF1`, caption "Modernisation"
  - Opex/BAU/enhancements → fill `#A9DCC6`, ink `#1E6F54`, soft `#E8F5EF`, caption "Business as usual · Enhancements"
  - Unmatched lanes draw sequentially from an 8-color fallback palette (the five above plus teal, olive, tan), skipping any color already used.
- Completed-feature trophy chip: white circular chip, navy icon `#1B2A5B`, small drop shadow. Bars for a feature completing in the *current* quarter get a solid `2px` navy (`#1B2A5B`) outline; bars with unresolved dates get a dashed orange (`#C2410C`) outline.
- **Swimlane colours mode** (Settings): Multicolour (default, the palette above) or **Monochrome**. Monochrome exposes a **Base colour** picker of 12 wheel hues (30° apart: Red 0°, Orange 30°, Amber 48°, Lime 90°, Green 120°, Teal 150°, Cyan 180°, Azure 205°, Blue 232°, Violet 262°, Magenta 290°, Rose 330°). In monochrome, lanes are coloured in **groups of three**: each group uses one base hue as 3 fixed-lightness tints — light `hsl(h,46%,86%)`, mid `hsl(h,46%,72%)`, deep `hsl(h,46%,58%)`; every next group of 3 advances to the following hue on the wheel starting from the picked one, wrapping `% 12` past the last. Tint order alternates per group (boustrophedon): even groups run light→mid→deep, odd groups run deep→mid→light. `ink` is `hsl(h,54%,28%)`, caption `soft` is `hsl(h,44%,95%)`. Mode + selected hue persist to localStorage.

## Layout (top to bottom)

**1. Header bar** (padded, bottom border):
- Left: eyebrow "PRODUCT DELIVERY"; title "Feature Roadmap" (31px bold); fiscal-year subtitle; meta line `{N} initiatives · {M} streams · 6 quarters · as of Q{n} {year}` (or "6 quarters · awaiting import" when empty).
- Right (only when data loaded): **filter chips**, two rows, right-aligned — SWIMLANE chips (toggle whole lane visibility) and STATUS chips (toggle by status value, including "No status"), each row with its own "Show all" reset link when any chip is off.

**2. Toolbar** (light `--surface-2` strip, bottom border) — left to right:
- **Download template**, **Import Excel** (navy filled label + hidden file input), **Export Data**, **Export Image**, **Clear** (text button), inline column hint text, status indicator (dot + message).
- Right-aligned **+ Add** button opening a small dropdown menu with "New Swimlane" and "New Feature" actions (click-outside / backdrop closes it).

**3. Timeline body** (the core):
- Relative container with an absolutely-positioned background layer holding: alternating-sprint light-blue column tints (Sprint view only, see below), a subtle shade over "next year / out of window" columns, and vertical gridlines at boundaries (a darker line at the year divider).
- A sticky header block stacks, top to bottom:
  1. **Year row** — two spans across a 6-column grid: current year across cols 1–4, "{nextYear} · PLANNING" across cols 5–6. Hidden while zoomed into a single quarter.
  2. **Quarter row** (shown when not zoomed) — 6 cells "Q1".."Q4","Q1","Q2" in IBM Plex Mono. The real current quarter is highlighted orange `#DE5B3C` with a "NOW" tag. Each cell (`.q-cell`) reveals a small zoom-lens button on hover (`.q-zoom-lens`, opacity 0→1 via CSS, not inline) — clicking it zooms the whole timeline into that single quarter.
  3. **Zoomed header** (shown when a quarter is zoomed) — flex row, quarter label on the left, a "Zoom out" pill button (navy, magnifying-glass-minus icon) on the **right**. This button (and any other pure-navigation chrome) carries `data-export-hide="1"` so it's stripped out of the html2canvas image export.
  4. **Month row** (Month view, or automatically inside zoomed-quarter view) — one cell per month, label + short caption.
  5. **Sprint row** (Sprint view only) — 42 sprint cells across the full 6-quarter span (7 sprints/quarter, numbered `.1`–`.7`, running Wed→Tue). When zoomed into one quarter, only that quarter's own 7 sprints are shown — never the neighboring quarter's last/first sprint. The current sprint is highlighted orange. Each cell's font size doubles when zoomed in (7px → 15px). Alternating sprint columns (odd index) get a very faint blue background tint (`rgba(96,140,230,0.045)`) for readability, computed against absolute sprint index so the pattern is stable across zoom/pan.
     - **Sprint end dates** (togglable in Settings, Sprint view only): each sprint cell shows its last day in `DDMMM` format (no separator, e.g. `14APR`) in a darker grey (`#5C6578`, orange when it's the current sprint), rotated -90° with `transform: rotate()` (not `writing-mode`, since that breaks html2canvas image export orientation) and right-aligned against the sprint's end gridline. When shown, the column's right-edge divider (inset box-shadow) extends up through the header so it visually reaches the top of the date text.
- **Empty state** (no data): dashed-border card, ↥ icon, "No roadmap data yet", instructions to import or download the template.
- **Swimlanes**: for each lane — a clickable header row (soft bg) with a chevron (rotates when collapsed), colored square, lane label (double-click to rename inline), caption, right-aligned count badge, and on-hover-only (`.lane-row-ctrl`) delete and drag-to-reorder-grip controls (both `data-export-hide="1"`).
  - Under each header, one **track row** per feature (alternating faint stripe). Each track holds an absolutely-positioned **bar**: left/width derived from `start/(end+1) × colWidth%` against whichever axis is active (quarter/month/sprint, zoomed or not). Bar uses lane fill/ink colors, rounded corners, soft shadow, and shows (in order): feature name (ellipsis), inline uppercase status text, and, right-aligned, a **trophy chip** — a small white circle with a navy trophy icon — shown when the feature's end date falls in the current quarter or any earlier quarter (i.e. it's done or wrapping up now). A feature ending in the *current* quarter additionally gets the solid navy outline described above (trophy alone signals "already completed"; outline + trophy together signal "wrapping up this quarter").
  - Bars support double-click-to-edit (inline name + status text inputs + delete), an unresolved-dates badge (`!`) when start/end couldn't be parsed, and overflow chips when a bar's start/end lies outside the current zoom window.
  - Left/right invisible 10px grip handles (`data-grip="l"`/`"r"`) for resize; bar body for move — see Drag section.

**4. Footer** (top border): a dismissible inline hint chip (left), and right-aligned **Recycle Bin** (with a red count badge when non-empty), **Settings**, and **Help** buttons — all icon + label, navy text/icons.

**5. Recycle Bin modal**: overlay + centered card. Navy header ("Recently deleted"). Body lists deleted swimlanes/features (colored dot, label, meta) each with **Restore** and permanent-remove (×) buttons. Footer-right "Empty Recycle Bin" link when non-empty. Empty state message otherwise.

**6. Settings modal**: overlay + centered card, navy header ("Preferences" eyebrow, "Settings" title). Body is a stack of rows (icon chip + title + description + a 2–3-way segmented control), separated by dividers:
  1. **Fiscal year start** — Jan or Apr (shifts which calendar months map to Q1 everywhere).
  2. **Timeline view** — Quarter / Month / Sprint (persisted; drives which header row(s) render and the bar-position math).
  3. **Sprint end dates** — Show / Hide (persisted). Both buttons are disabled/greyed when Timeline view isn't Sprint (dates only make sense there).
  4. **Density** — Comfortable / Compact (row height, bar height, font size).
  5. **Theme** — Light / Dark.
  All choices persist to localStorage and re-apply on load.

**7. Help modal**: overlay + centered card, navy header. Six sections (icon chip + title + bulleted steps): Get started, Columns in your file, Reading the timeline, Reschedule by dragging, Filter & focus, Export & share. Footer note on data locality and export options.

## Data model & parsing
Each item: `{ lane, name, ids, s, e, status, note }` where `s`/`e` are integer **month indices 0–17** internally (Jan of base fiscal year .. Jun of the year after next), derived from/rendered as quarters, months, or sprints depending on view. Legacy saved data with quarter indices 0–5 is upconverted.

**Import parsing** (first worksheet → JSON rows):
- Fuzzy-match column headers by regex: name, swimlane, start, end, key/ticket, status. Also detect per-quarter boolean columns named like "Q1", "Q2"…
- Skip rows with blank name. Blank swimlane → "Uncategorised".
- **parseQuarter("Q1 2026")**: extract Q-number 1–4 and 4-digit year, respecting the fiscal-year-start setting; clamp to the visible window. Derive missing start/end from ticked quarter columns; fill one from the other; swap if end < start.
- **ID extraction**: pull ticket-style patterns into a `#1234`-style string; strip bracketed IDs from the displayed name.
- Friendly error on unparsable files.

## Drag-to-reschedule
Delegated `pointerdown` on the timeline foreground node (re-bound on re-render). Detect left grip (resize start), right grip (resize end), or bar body (move). On `pointermove`, convert pixel delta to whole-column delta at the current zoom granularity (quarter/month/sprint width):
- **move**: shift both s and e, preserving span, clamped to the valid range.
- **l**: move start, clamped between 0 and current end.
- **r**: move end, clamped between current start and max.
Live-preview the dragged bar (raised z-index, stronger shadow, grabbing cursor, no transition). Commit + persist on `pointerup`.

## Zoom
Hovering a quarter cell reveals a lens button; clicking it narrows the whole timeline to that quarter's own months/sprints (never bleeding into the adjacent quarter's edge column) and swaps the header to the "zoomed" row (quarter label left, "Zoom out" button right). Sprint/label font sizes scale up in this state for legibility.

## Swimlane & feature management
- **+ Add** menu: create a new empty swimlane or a new feature (added to the first/last lane) inline, immediately editable.
- Lane header: double-click label to rename; hover reveals delete (moves to Recycle Bin) and a drag grip to reorder lanes.
- Feature bar: double-click to edit name/status inline, or delete (also recoverable from Recycle Bin).
- Recycle Bin stores soft-deleted lanes/features until restored or permanently cleared.

## Persistence & state
- localStorage keys: items+filename, density, fiscal-year-start, timeline-view, sprint-end-dates toggle, theme — each read on mount and written on change.
- State fields include: `items, fileName, status, statusType, collapsed{}, hidden{}, hiddenStatus{}, helpOpen, settingsOpen, trashOpen, addMenuOpen, editingLane/Feature, drag, zoomQ, theme, density, fyStart, timelineView, showSprintDates, trash[]`.

## Export behavior
- **Export Image** (html2canvas, `scale:2`, white bg) hides the toolbar and any element flagged `data-export-hide="1"` (zoom-out button, lane hover controls) before capture, then restores them — so the exported image shows only the read-only board chrome.
- **Export Data** writes current items back to `.xlsx` in the same template column format.

## Behavioral details to preserve
- Filtering by status recomputes lane counts to reflect only visible items.
- Lanes are built in order of first appearance in the data (then reorderable by drag).
- Quarters/months/sprints are anchored to the current real-world date and fiscal-year-start setting; "now" highlighting is computed live at every granularity.
- All exports guard against CDN libs not yet loaded, surfacing a "…engine still loading — try again" status.
- Bar position math uses a single column-width percentage constant per active granularity (quarter/month/sprint, zoomed or not); gridlines, shading, and bars all derive from the same constant.
