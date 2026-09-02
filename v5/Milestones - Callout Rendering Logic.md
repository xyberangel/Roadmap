# Milestone Callout Rendering — Reference

How milestone diamonds and their name labels are laid out in the **Gantt** layout.
Source of truth: `_unpack/live.html` (and the bundled copy in `index.html`), inside the Gantt
row builder — search for `--- milestones: diamonds on the bar`.

---

## 1. Vocabulary

| Term | Meaning |
| --- | --- |
| **Diamond** | The 9px rotated square marking a milestone's date. Rides the **top edge** of the plan bar. |
| **Label** | The milestone name, 11px, drawn in the **callout band above the bar**. |
| **Vertical leg** | 1px line at the diamond's `cx`, from the bar's foot up to the label's line. |
| **Horizontal leg** | 1px line on the label's line, running through the **gap** between the label's near edge and `cx`. Never crosses text. |
| **Line** (`lane` / `glane`) | One 12px row of the callout band. Line 0 is closest to the bar; lines climb upward. |
| **Corridor** | The horizontal interval a label occupies **including its leg**: `[min(cx, left), max(cx, left + w)]`. A corridor therefore always contains its own `cx`. |
| **Band** | The block above the bar holding all lines. Height = `lines × 12 + 2`. |

Geometry constants: `GAP = 6` (label hugs its own diamond), `PAD = 6` (clearance between two
corridors on one line), diamond de-crowding `13px`, line pitch `12px`.

---

## 2. Pipeline

1. **Collect** — `featureMilestones(it)`, already date-sorted by `mutateMilestones`.
2. **Project** — `cx = dateToX(date)`. Milestones outside the track are dropped.
3. **De-crowd diamonds** —
   - identical date ⇒ identical `cx` (they sit exactly on top of each other, deliberately);
   - different dates closer than `13px` ⇒ the later one is nudged to `prev + 13`.
4. **Pack labels** — only when milestone display is `names` (§3).
5. **Size the band** — `msLanes` = highest line used + 1; `bandH = msLanes * 12 + 2`; the row's
   height becomes `rowH + bandH`, and the bar plus its diamonds shift down by `bandH`.
6. **Assign y** — a single pass converts each mark's `glane` into real `top` values for label,
   horizontal leg and vertical leg. **The packer never sets y**; it only decides line + x.

---

## 3. The packer

### 3.1 Fill order

Lines fill **from the bar upward**. Line 0 is swept left→right and takes every label that fits;
whatever is refused defers to line 1, and so on. A label therefore climbs only when the space
below it is genuinely taken.

> **Historical note.** The previous packer walked latest→earliest and let lines climb but never
> descend (`fromLane = max(0, prevLane)`). That single constraint — not the right-hand bias — is
> what made bands tall: one crowded pair pushed every earlier label upward. Removing it took a
> representative 8-milestone row from 7 lines to 4.

### 3.2 Side preference, in precedence order

1. **The earliest milestone always reads to the RIGHT** of its diamond. It climbs a line rather
   than taking the space in front of itself — it is the band's anchor.
2. **Same-date alternation.** If the current line already carries the *right-hand* label of this
   exact `cx`, this label tries the **left** first, so a same-date run reads
   right, left, right, left and opens a line only every second label.
3. **Otherwise right first**, and left only if the right side fails — i.e. a left placement is
   allowed only when it saves a line.

### 3.3 Legality — two conditions, checked for every candidate

A candidate at `left = lx` with corridor `[a, b]` on line `L` is legal iff:

1. **Disjoint on the line.** For every corridor `o` already on line `L`:
   `b + PAD <= o.a  ||  a >= o.b + PAD`.
   **Exception:** if `o.cx == cx` and `o.side != side`, the two are the same-date twins — they
   share one diamond and one vertical leg, so their corridors are allowed to meet **at the
   diamond and nowhere else**.
2. **No corridor spans a higher diamond.** `a + 3 < c < b - 3` must hold for no `c` in the set of
   milestones not yet placed. Those land on higher lines and their vertical legs would rise
   through this label's text. (The strict inequality is what lets same-`cx` twins pass, since
   their corridors start exactly at `cx`.)

Together these make crossings impossible without the old date-order guarantee. Two-sided packing
is legal because a leg only ever travels **below** its own line: verticals pass through lower
lines, never higher ones.

### 3.4 What the rules cost

A line can read **out of date order** — a later milestone's label may sit to the left of an
earlier one's. The horizontal leg is the only cue to ownership. This was accepted deliberately in
exchange for the shorter band.

### 3.5 Overflow

- Bands are **uncapped**: a row grows as tall as it needs. Nothing is ellipsised and no label is
  ever dropped (both behaviours were removed).
- A name wider than the entire track can never fit any line. It gets a fresh line of its own,
  starts at its diamond, runs to the track edge and **clips** there.
- Names longer than 24 characters are ellipsised by `clipMs` — **except** under Scale to Fit.

### 3.6 Scale to Fit (`state.fitPrev` set ⇒ `msNoClip`)

Scale to Fit exists to show the data in full, so while it is on:

- the 24-character `clipMs` cap is lifted — full names render;
- the track-edge `maxWidth` clip is dropped (`overflow: visible`);
- a candidate may **overhang the right edge** instead of being refused a line.

---

## 4. Colour and state

`milestoneState(m)` ⇒ `planned` | `done` | `late`.

| State | Colour (light / dark) | Diamond |
| --- | --- | --- |
| Planned | bar colour | hollow |
| Done | `#3F7D55` / `#7FB893` | solid |
| Late | `#B87514` / `#E2A33C` | solid |

Labels use `var(--text-2)`, except **late**, which takes the amber. Legs are the milestone's
colour at `0.28` opacity. Every diamond carries a tooltip: `name · date · state`.

---

## 5. Re-render on mutation

`addMilestone`, `deleteMilestone` and any date change funnel through **`mutateMilestones`**, which
clones → mutates → clamps to the bar's span → sorts by date → persists as one undo step, then:

- clears the text-width memo (`this._msW = {}`),
- bumps `state.msNonce`.

The nonce is carried on the Gantt row so the whole band is re-packed from scratch rather than
patched in place. Packing is **live during a bar drag** — labels track their diamonds.

---

## 6. Downstream consumers

The packer publishes per-mark geometry that other code reads; keep these fields when editing:

| Field | Meaning |
| --- | --- |
| `glane` | line index, `null` ⇒ not rendered |
| `gx`, `gcy` | diamond centre |
| `glabelX`, `glabelY`, `glabelW` | label box |
| `gname`, `gcol`, `gsolid` | text and colour |

**PPTX export** (`// Milestones: diamond, elbow legs, name`) redraws from these fields, so it
follows any packer change automatically — no parallel layout code exists.

**Status stamp interaction:** if any diamond falls in the in-bar status stamp's zone, the stamp is
dropped rather than overlapped.

---

## 7. Not implemented

Band-height **compression when the chart would otherwise scroll**. Decided but deferred: it
touches total chart height, Scale to Fit and the PPTX page sizing. Bands are uncapped today.
