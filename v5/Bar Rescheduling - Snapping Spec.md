# Feature Bar Rescheduling — View-Dependent Snapping Spec

## Epic
A user can click-drag a feature bar to move it, or drag either edge to resize it. The
granularity that the bar's **start** and **end** dates snap to is governed **entirely by
the active timeline view toggle** (Quarter / Month / Sprint). Snapping applies identically
to move and to resize.

### Global rules (all views)
- Snapping occurs live during the drag (visual) and is committed on pointer-up (persisted).
- Move preserves the bar's span; the left edge snaps the start only; the right edge snaps
  the end only. An edge may never cross the opposite edge.
- All snapping is clamped to the roadmap's bounds (first day of the first quarter → last
  day of the last quarter).

---

## Scenario 1 — Quarter View
**User story:** As a planner in Quarter view, when I reschedule or resize a feature, its
dates snap to whole standard quarters so the bar always fills complete quarter columns.

**Acceptance criteria**
- **GIVEN** the view toggle is "Quarter", **WHEN** I move or resize a bar, **THEN** the
  start snaps to the **first day** and the end to the **last day** of a standard quarter:
  - Q1: **Jan 1 – Mar 31**
  - Q2: **Apr 1 – Jun 30**
  - Q3: **Jul 1 – Sep 30**
  - Q4: **Oct 1 – Dec 31**
- **Move** snaps to whole quarters (3-month step); a moved bar never starts or ends mid-quarter.
- **Resize (either edge)** snaps in **1-week (7-day) increments** — sub-quarter sizing is
  allowed directly in Quarter view (no need to switch to Month/Sprint). Each drag step
  grows or shrinks the bar by exactly 7 days, day-precise.
  - Minimum bar length on resize is **one week**.
  - A resized bar stores explicit day edges (`dS`/`dE`) and renders day-precise; whole-quarter
    bars still align exactly to the quarter gridlines.
- **Constraint:** dates must **NOT** align to sprint boundaries. Any previously stored
  sprint pin (`spS`/`spE`) is cleared on commit.
- Export writes the exact day edges for week-sized bars, or calendar quarter boundaries
  (Jan 1 – Mar 31 etc.) for whole-quarter bars.

> Note: 7-day resize granularity is measured from the bar's current edge. Day-precise
> rendering applies in the normal (unzoomed, unfiltered) timeline; under quarter-zoom or a
> quarter filter the bar falls back to column alignment, though its stored dates and export
> stay exact.

---

## Scenario 2 — Month View
**User story:** As a planner in Month view, when I reschedule or resize a feature, its
dates snap to whole calendar months.

**Acceptance criteria**
- **GIVEN** the view toggle is "Months", **WHEN** I move or resize a bar, **THEN** the
  start snaps to the **first calendar day** of a month and the end to the **last calendar
  day** of a month (e.g. Feb 1 – Feb 28/29).
- Snap step = **1 month**; a bar can never start or end mid-month.
- **Constraint:** dates must **NOT** align to sprint boundaries. Any previously stored
  sprint pin (`spS`/`spE`) is cleared on commit; dates are derived from month indices.
- Minimum span is one full month.

---

## Scenario 3 — Sprint View
**User story:** As a planner in Sprint view, when I reschedule or resize a feature, its
dates snap to whole sprints and pin to those sprints' exact calendar dates.

**Acceptance criteria**
- **GIVEN** the view toggle is "Sprints", **WHEN** I move or resize a bar, **THEN** the
  start snaps to a sprint's **start date (Wednesday)** and the end to a sprint's **end
  date (Tuesday)**.
- Snap step = **1 sprint column**; a bar always spans whole sprints (never a partial sprint).
- The occupied sprint columns are **pinned** on commit (`spS`/`spE` stored), so the exact
  start/end dates are fixed to those specific sprints — not re-derived from month boundaries.
- The pinned sprint span is mirrored back to month/quarter indices so Month and Quarter
  views (and Export) stay consistent.

---

## Cross-view note
Switching from Sprint to Month/Quarter and then editing a bar drops its sprint pin (a
month/quarter edit overrides sprint placement); editing in Sprint view re-establishes it.
This is intended and preserves the "no sprint alignment in Quarter/Month" constraint.

---

## Implementation notes (as built in `index.html`)
- **Snapping** — `beginDrag` sets `step = isMonthView() ? 1 : 3`; the sprint branch snaps
  to whole sprint columns. `onDragMove` applies the snap; `onDragEnd` commits.
- **Sprint pin lifecycle** — `onDragEnd`: Sprint-view edits store `spS`/`spE`; Month/Quarter
  edits `delete` them.
- **Date derivation** — `featureSprintRange(it)`:
  - `spS != null` (sprint-pinned) → returns the sprint's `start` (Wed) / `end` (Tue).
  - otherwise (Month/Quarter-placed) → returns **calendar** boundaries
    `monthStartDate(ms)` … `monthEndDate(me)` (first day of start month, last day of end
    month), never sprint edges. This is what Export writes out.
