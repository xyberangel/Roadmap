# Presentation Mode — Implementation Plan

Goal: one keypress turns the board into something you can put on a meeting-room screen —
no chrome, larger type, and a way to walk the audience through it theme by theme.
Nothing about the data model changes; this is a render mode plus a small navigation state.

## 1. Entry and exit

- New toolbar button `▶ Present` (next to Help), and `P` as a shortcut when focus isn't in a field.
- `Esc` exits. Browser fullscreen is requested on enter (`requestFullscreen` on the card element)
  and released on exit; if the request is refused, the mode still works as an in-page takeover.
- State: `presenting: false, presentStep: 0`. Deliberately **not** persisted — nobody wants to
  reopen the app in presentation mode.

## 2. What the mode changes

Chrome hidden: header block, toolbar, view/filter menus, footer, hint chip, status line,
the ⚑ chips, and every edit affordance (bar drag grips, date cells become plain text,
inline editors close on enter).

Chrome kept: a slim bottom bar — step label, `‹ ›` arrows, step dots, and an exit `✕`.
It fades out after 3s of no pointer movement and returns on move.

Scale: the card goes edge-to-edge, `border-radius: 0`, no page padding. Type steps up one
notch across the timeline (row labels 11.5→15px, month headers 10→13px, milestone
callouts 11→14px, status stamps 8.5→11px, row height 24→34px). Implement as a single
`pScale` factor threaded into `buildGanttModel` rather than a second set of literals —
the model already derives every size from `rowH` and a handful of constants, so one
multiplier keeps the two paths honest. `ganttCacheKey` must include it.

## 3. Stepping

Steps are derived, not authored:

1. **Overview** — the whole board, all themes, full date range.
2. **One step per visible theme** — that theme's rows only, other themes dimmed to 12%
   opacity (not removed, so the timeline geometry never moves between steps).
3. **Optional last step: "At risk"** — only if any feature has a late milestone; shows just
   those rows. Skipped entirely when there are none.

Navigation: `→`/`Space`/click-right-half advances, `←` goes back, `Home` returns to overview,
number keys jump to a theme. Dimming animates over 220ms; the axis never re-scales, so the
eye keeps its place.

## 4. Reuse, not a fork

- The timeline comes from the existing `buildGantt` model — presentation mode passes
  `showStatus`/`showDates`/etc. from the same View settings the user already chose.
- The theme-dimming is one extra field per row (`dim: true|false`) consumed by
  `trackRowStyle`/`nameCellStyle` opacity. No second render path.
- Gallery and Spotlight get the same treatment for free: they already render from
  `LANES`/`itemsByLane`, so the step list is "overview + one per lane" there too.

## 5. Cost and order of work

| Step | Work | Notes |
|---|---|---|
| 1 | `presenting` state, enter/exit, fullscreen, chrome hiding | half day |
| 2 | `pScale` through the Gantt model + cache key | half day |
| 3 | Step derivation, dimming, keyboard nav, bottom bar | one day |
| 4 | Gallery/Spotlight step support | half day |
| 5 | Polish: cursor auto-hide, 220ms transitions, dark-theme check | half day |

## 6. Open questions

- Should presentation mode force a fixed window (e.g. 12 months) so the board can't be
  presented at an unreadable "All" span? Yes, snap to the widest span
  that keeps month labels ≥12px, and say so in the bottom bar.
- Do you want a per-step note field (a sentence you read out), stored per theme? No
- Second-screen support (notes on the laptop, chart on the projector) is a much bigger
  build — worth deferring until the single-screen mode has been used in a real meeting.Not now.
