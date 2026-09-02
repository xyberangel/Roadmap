# Milestones — Implementation Plan

Goal: every feature can carry one or more milestones. They render as diamonds on the feature
bar with elbow-line callouts naming them, in a **new "Milestones" layout** that sits beside
Gallery / Spotlight / Gantt. Rows with milestones can take double height so names never
overlap.

## 0. The Milestones layout
A fourth layout button: `Gallery · Spotlight · Gantt · Milestones`, i.e.
`onLayoutMilestones` → `layoutMode: 'milestones'`, persisted under the existing `LAYOUT_KEY`.

- It is the Gantt timeline plus the milestone furniture — same swimlanes, bars, granularity
  segments (Weeks/Sprints/Months/Quarters), span picker + ‹ ›, dates, now-line, drag/resize.
  Implemented as `isMilestones` alongside `isGantt`, with both flowing into the same
  `buildGantt` (one extra argument), so there is no second timeline engine to maintain.
- Everything milestone-specific — the display-mode control, doubled rows, diamonds, callouts,
  the row popover, the legend key — lives only in this layout. The Gantt layout is untouched:
  same code path, milestone drawing switched off, so no regression risk there.
- Gallery and Spotlight ignore `ms` entirely: the data is preserved, just not drawn.

**Display mode** (replaces the earlier toolbar flag; a 3-way segment in this layout's toolbar,
key `roadmap_milestone_display_v1`, default **Diamonds**):

| Mode | What it shows | Vertical cost |
| --- | --- | --- |
| **Diamonds** | Markers on the bars only; names on hover/popover | none — rows keep their density height |
| **Names** | Diamonds + elbow callouts with names | rows *with* milestones double |
| **Off** | Nothing (bars only) | none |

Default Diamonds means opening the layout always shows something without the chart jumping
taller; Names is the deliberate "I'm presenting this" step.

**Build order** — three commits, each shippable alone:
1. Data + editing + Excel (§1, §3, §7) — milestones exist, are editable, round-trip.
2. The layout: button, display modes, per-row heights, diamonds, callouts (§0, §4, §5).
3. Date propagation + PPTX (§6, §8).

## 1. Data model
- Add `ms:[{ id, name, d, st }]` to a feature item (`d` = ISO date, `id` = short uid unique
  within the feature — also the React key for popup rows and chart marks, `st` = status, §5b).
- Persist with the existing items store, so it rides along with import/export, undo/redo
  (`persistItems`), localStorage, and delete → Recycle Bin → restore with no new key.
- **The feature's span** is `ganttFeatureSpan(item)`, not `dS..dE` directly: month-precision
  imports have only `ms`/`me`, so the span falls back to
  `monthStartDate(ms)..monthEndDate(me)`. Every clamp, seed and shift below means that span.
- Validation on write: clamp `d` into the span; sort by date; trim names and drop entries
  whose name is empty after trimming.

## 2. Discoverability
Milestones must be visible as *data*, not only as chart decoration:
- **Count chip** in the feature's left-panel row (`⚑2`), shown in the Milestones layout in
  every display mode. Click → opens the feature editor scrolled to the Milestones block.
- **`+⚑` on row hover** → adds a milestone and focuses its name field. This is the primary
  add affordance; the popup button is the secondary one.
- **Shift-click a bar** at a point → adds a milestone at that date.
- **Click a diamond** → opens the editor with that milestone's row focused. Diamond hit area
  is a ~16px transparent box around the 9px glyph.
- The layout's empty state (no feature has a milestone yet) says how to add one, rather than
  disabling controls.

## 3. Add / edit / delete
All editing lives in the existing feature editor popup (`openFeatureEditor` — double-click a
bar or the ✎ in the row), which already holds Name / Status / Start / End. No new modal.

The popup gains a **Milestones** block under the date fields:

| Action | How |
| --- | --- |
| **Add** | "+ Milestone" button, or `+⚑` on the row, or shift-click the bar. Seeds name `"Milestone N"`, date = bar midpoint (or the clicked date), status `planned`, and focuses+selects the name field. |
| **Edit name** | Text input per row, commits on blur/Enter. **Enter commits and adds the next milestone** (Esc closes) so five milestones are one typing flow. |
| **Edit date** | The same date control the Start/End fields use — typing, arrows and sprint-snap all behave identically. Clamped to the feature's span. |
| **Edit status** | 2-way toggle per row: Planned / Done (see §5b — "at risk" is derived, not entered). |
| **Delete** | × at the right of the row; immediate, undo restores it. |
| **Reorder** | None — rows always display in date order. |

- The block is capped at ~5 visible rows with internal scroll, so the popup can't run off
  screen on a feature with many milestones.
- Handlers alongside the existing `onFeat*` set: `onMilestoneAdd(key, iso?)`,
  `onMilestoneName`, `onMilestoneDate`, `onMilestoneStatus`, `onMilestoneDelete`.
- Each clones items, mutates that feature's `ms`, re-sorts, clamps, then calls
  `persistItems` — one undo step per action, persisted exactly like a date edit.
- Dragging a diamond along the bar to change its date: v2 follow-up (click-to-edit covers it).

## 4. Row layout in buildGantt
Row height becomes per-row instead of the constant `rowH = 24`:
- `rowH(item) = base * (mode === 'names' && visibleMilestones(item).length ? 2 : 1)`, where
  `base` is the current density-derived row height — never a hard-coded 24.
- Build a per-row table `[{ key, top, h }]` in one top-down pass with a running `y` cursor;
  every consumer that assumes `i * rowH` (bars, row lines, lane bands, drag hit-testing,
  now-line height, track height, PPTX row mapping) reads `top`/`h` from it. One source of
  truth shared by screen and exports.
- Collapsed lanes and status-filtered rows are excluded before the pass; heights always match
  what is drawn.
- Height counts only milestones inside the visible window, so a scrolled-away milestone never
  leaves an empty band.
- Diamonds sit on the bar centre line; the extra half-row below the bar is that row's callout
  band.

## 5. Diamonds + elbow callouts
- Diamond: 9px rotated square at `dateToX(milestone.d)`, clamped to the track, above the bar
  (over the bar, under the now-line); fill/stroke per status (§5b); `title` =
  `name · date · status`.
- Callout: elbow polyline — down from the diamond, then horizontal to the label — drawn as
  two 1px absolutely-positioned divs (no SVG) so it survives PPTX shape export.
- Label: 11px minimum, `--text-2` (not the bar colour — the diamond already carries the
  colour coding), `white-space: nowrap`, in the callout band.

### Overlap avoidance (per row, deterministic)
1. Measure label widths once with a cached `canvas.measureText` at the label font, ×1.1
   safety factor (character-count estimates drift on a proportional face).
2. Sort the row's milestones by x, walk left→right placing each label at its diamond x; if it
   would overlap the previous label's right edge + 6px gap, shift it right.
3. If a shifted label would exit the track, flip it to the left of its diamond and re-walk
   right→left for the tail.
4. Alternate between two callout lanes (upper/lower half of the band) when a row has more
   than 3 milestones or a collision survives step 3 — the elbow's vertical leg length
   distinguishes the lanes, so lines never cross labels.
- Milestones outside the visible window are not emitted at all.

### Row popover (replaces a "+N" truncation)
Hovering (or focusing) a row's callout band opens a small popover listing **every** milestone
for that feature in date order — name · date · status. It is the read-out for the Diamonds
mode, and the escape valve for a row with more milestones than the band can label, so no
milestone is ever hidden behind a `title` attribute.

### 5b. Milestone status
Two stored values, one derived:
- `planned` — hollow diamond, bar-colour stroke.
- `done` — solid diamond in the bar colour, label prefixed with a check.
- **`late` (derived, never entered)** — `planned && d < today` renders with a 2px amber
  stroke and an amber label. Derived means it can't go stale; a manual `risk` value stays
  supported for imports as an override.
Missing/unknown falls back to `planned`. Colours come from the design system's status
palette — no new colours invented. The legend gains a milestone key (◇ planned · ◈ late ·
◆ done) in this layout only.

## 6. When feature dates change
One rule set, applied wherever a feature's dates change — bar drag, resize, the popup's
Start/End fields, table cells (all of which funnel through `applyFeatureDates` or the drag
commit).

**Start changes — the offset rule**
- Each milestone keeps its offset from the feature start, so the set shifts by the same
  whole-day delta `Δ = newStart − oldStart`. Start 1 Oct → 8 Oct with a milestone on 14 Oct ⇒
  21 Oct; pulling the start earlier shifts them earlier.
- Same behaviour from drag, popup field or table cell — a start edit is always a relative
  shift, never a clamp.

**Whole-bar move (drag)** — start and end move together, so the offset rule already gives the
right answer: same delta, position within the bar unchanged.

**End changes (resize from the right)** — start is unchanged, so offsets still hold: dates
stay put, then clamp. A milestone beyond the new end sticks to the end day rather than being
dropped, and the popup shows it there so the user can fix or delete it deliberately.

**Both change at once (import, paste, span edit)** — offset rule against the new start, then
clamp to the new end.

**Implementation notes**
- Single helper `shiftMilestones(item, oldStart, newStart, newEnd)` called from every date
  mutation path, so none can forget it; returns a new `ms` array.
- Runs inside the same items clone + `persistItems` commit as the date change → one undo step
  restores bar and milestones together.
- Sprint-snapped drags pass the *snapped* start, so milestones never land mid-snap.
- Day arithmetic on local-midnight dates (the same `DAY` constant the drag code uses), so DST
  can't move a milestone across a day boundary.

**Deliberately not doing:** proportional rescaling of milestone spacing when the bar's length
changes — it moves milestones the user never touched and makes a resize non-reversible.

## 7. Excel export / import round-trip
`onExportExcel` writes one sheet, `Roadmap`
(`Feature Name | Theme | Start | End | Status | Category | Description`). Milestones ride
along as **one extra column on that same sheet** — no second sheet, no change to file shape.

**The `Milestones` column**
- Appended last. One cell per feature, all its milestones semicolon-separated in date order:
  `Beta cutover@2026-10-14!done; GA sign-off@2026-11-25`.
- Grammar: `name@YYYY-MM-DD` with optional `!status` (`planned` | `done`, plus `risk`
  accepted for legacy files); missing status = `planned`. Spaces around `;` ignored.
- Names containing `@`, `;` or `!` are wrapped in double quotes on write; the parser strips
  one layer of quotes and splits on the last `@`.
- Empty for features without milestones, so such boards export as they do today.
- Column width `{ wch: 46 }` appended to `ws['!cols']`.

**Import (same column, same parser)**
- `parseWorkbook` picks up a header matching `/^milestones?$/i` and splits with the grammar
  above; each entry is clamped into the feature's span and given a fresh `id`.
- Unparseable fragments and clamped dates are skipped/adjusted and counted into the existing
  import warning toast rather than failing the import.
- Round-trip guarantee: Export → edit in Excel → Import reproduces the same board.

**Other touch points**
- **Download template** (`onTemplate`) gains the column with one example row, so the syntax is
  self-documenting.
- Export toast becomes `Exported feature-roadmap.xlsx · N features · M milestones`.
- Help panel: one line in "Export & share" and "Get started" about the column syntax, plus a
  short "Milestones layout" entry describing the display modes.

## 8. Exports (image / PowerPoint)
- Image export: nothing beyond the layout being real DOM.
- PPTX: a diamond shape (`addShape('diamond')`), two thin lines for the elbow, and a text box
  per label, reusing the existing `TX()` / per-row mapping — filled vs outlined per status.
  Row heights come from the same per-row table, so the slide matches the screen.
- Exports follow the current display mode (Names exports names, Diamonds exports markers
  only) and the visible window, as they already do.

## 9. Edge cases to settle in code
- **Zero-length feature** (start = end): all milestones land on one day; the overlap pass
  stacks their labels across the two callout lanes, popover carries the rest.
- **Very crowded row**: labels that can't be placed are dropped from the band (never
  overlapped) and remain listed in the row popover.
- **Feature deleted / restored from the Recycle Bin**: milestones travel with the item.
- **Switching layouts** (Gantt ↔ Milestones): same dates, same window anchor, same span — the
  view swaps furniture only.

## 10. Test checklist
- Layout button: Milestones persists across reload; Gantt renders exactly as before.
- Display modes: Diamonds never changes row height; Names doubles only rows that have
  milestones in the visible window; Off matches Gantt pixel-for-pixel.
- Two milestones 1 day apart → labels offset, elbows readable, nothing clipped; one at each
  end of the track → labels flip inward.
- Weeks / Sprints / Months / Quarters: diamonds land on the right date in each.
- Drag a bar → milestones shift by the same delta; edit Start in the popup and in a table cell
  → same relative shift; resize the end → they stay put and clamp; one undo reverts bar +
  milestones together.
- Add via `+⚑`, shift-click and the popup button; Enter-chains five milestones; delete + undo.
- `late` derivation flips at midnight without an edit; all states legible in light and dark.
- Excel: export with and without milestones, re-import → identical board; a hand-typed cell
  with odd spacing and no `!status` imports; a name containing `;` survives escaping.
- Image + PPTX in both Diamonds and Names modes.

## Open question
Milestone name length cap — 24 chars keeps callouts readable at Quarters zoom; longer names
would need truncation with the popover as the full read-out. Confirm the cap, or allow
wrapping onto two lines inside the doubled row.
