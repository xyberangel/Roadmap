# UX Improvements — Implementation Plan

Source of truth: `_unpack/template.html`. After edits, rebundle to `index.html` via super_inline_html
(the 3 runtime UUID scripts + React/ReactDOM live in `_unpack/`; keep them). Line numbers below
are approximate (~v1787127878814814) — re-grep before editing.

## #1 — Consolidate index filtering (remove duplicate)
**Problem:** indexes can be filtered from BOTH the top "INDEXES" row chips (template ~271–289,
`indexChips`) AND a duplicate INDEX chip grid inside the Filters dropdown (~162–197,
`indexFilterChips`). Two homes = confusion.
**Plan:**
- Make the top INDEXES row the single home for filtering + editing (already supports click=toggle,
  dblclick/✎=edit).
- In the Filters dropdown, DELETE the `indexFilterChips` grid + its "Index" header/clear button
  (~162–188). KEEP only the "Show indexes" toggle (~190–197) and add one line of helper text:
  "Filter and edit indexes in the Indexes row above the timeline."
- Leave `indexFilterChips` in renderVals or remove it too (grep for `indexFilterChips` usages first;
  safe to delete the value if no longer referenced). Keep `clearIndexFilter`, `indexChips`,
  `onToggleIndexBar`.
- Verify: opening Filters no longer shows the chip grid; top-row filtering still works.

## #3 — Edit affordance (discoverability of double-click edit)
**Problem:** double-click a bar to edit, double-click a theme title to rename — no visible hint.
**Plan:**
- Feature bars: add a small pencil glyph (✎) that appears on bar hover, right-aligned inside the bar
  (near the status). Bar markup ~440–460 (the `notEditing` branch, `row.name` span). Add a
  `<span>` with `style-hover`-driven opacity on the bar container, `title="Double-click to edit"`,
  clicking it opens the editor (reuse the same setState as the dblclick path in the pointerdown
  handler ~1455 — extract an `openFeatureEditor(key, barEl)` method and call from both).
  Hide it during export (`data-export-hide="1"`) and while editing/dragging.
- Theme titles: the lane header (~370–410) — add a faint ✎ next to the label on lane hover with
  `title="Double-click to rename"`. Reuse existing rename entry point.
- Keep affordances subtle (opacity ~0 → ~0.55 on hover) so they don't add visual noise.

## #6 — Accessibility (focus + non-colour cue)
**Problem:** themes/indexes distinguished by hue only; bars/chips lack visible keyboard focus.
**Plan:**
- Focus rings: add `style-focus` (outline: 2px solid var(--ink); outline-offset:2px) to: feature
  bars, index chips (top row + edit popover), theme chips, toolbar buttons, date inputs. In this DC,
  pseudo-states are `style-focus="..."` attributes (NOT CSS classes).
- Keyboard access to bars: give each bar `tabindex="0"` and an `onKeyDown` that opens the editor on
  Enter/Space (route through the shared `openFeatureEditor`).
- Non-colour cue for indexes: index dot already differs (square vs circle for None). Reinforce by
  keeping the index NAME always visible on the chip (already true) and, when an index filter is
  active, add a thin left border / different border-style on matching bars in addition to colour
  (bar `barStyle` ~2555; add `borderLeft` accent when `rowIndex` present) so grouping survives
  greyscale.
- Do NOT introduce new colours; use existing tokens.

## #7 — Clarify top-right chip × meaning
**Problem:** two chips top-right — a metadata chip ("18 features · 4 streams · 6 quarters · as of…")
and an import chip ("Imported … .xlsx · 18 features") — both render an "×" that reads as
"remove filter". Grep for these near the header (search "features ·" / "Imported ").
**Plan:**
- Metadata chip: REMOVE its × entirely (it's informational, not dismissible).
- Import chip: KEEP the × but make its intent explicit — `title="Clear imported file info"` (or
  whatever it actually does; confirm the handler). If it only clears the label, consider relabeling.
- Confirm each chip's onClick before changing; don't break existing clear behaviour.

## #9 — Empty-lane prompt
**Problem:** a theme with 0 features shows blank space + a "0" count.
**Plan:**
- In the lane render (LANES.map ~2588+), when `items.length === 0` and not collapsed, render a faint
  full-width row inside the track: dashed 1px border, `color:var(--text-faint)`, text
  "No features yet — + Add a feature, or drag one here." Reuse the empty-state styling vocabulary
  (see existing `[data-empty-state]`, board-level empty state ~grep "data-empty-state").
- Mark it `data-export-hide="1"` so exported images stay clean.
- Ensure it doesn't interfere with drag-drop onto the lane.

## #10 — Footer discoverability (surface Settings/Help)
**Problem:** Recycle Bin / Settings / Feedback sit in the footer (below the fold on tall boards).
Footer markup ~503–520.
**Plan:**
- Add a Settings gear button to the top toolbar next to Help (toolbar ~top of template, near the
  Help button — grep `onOpenSettings` / "Help"). Reuse `onOpenSettings`.
- Keep the footer copies too (no harm), OR make the footer `position:sticky; bottom:0` with a
  translucent surface bg so it's always reachable. Prefer surfacing Settings in the toolbar (lower
  risk) unless the user wants the sticky footer.
- Recycle Bin already has a badge; consider mirroring it near the toolbar only if it has items.

## Suggested order
#7 (trivial) → #10 (toolbar Settings) → #1 (delete dup grid) → #9 (empty lane) →
#3 (edit affordance, shared openFeatureEditor) → #6 (focus/keyboard/non-colour, builds on #3).

## Verify each
Rebundle, show_html, open Filters + a bar editor + an empty lane, tab through for focus rings,
then ready_for_verification.
