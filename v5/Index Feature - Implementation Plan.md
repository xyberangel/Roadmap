# Index Feature — Implementation Plan

## Goal
Let users define named, colored **Indexes** (think labels/categories), assign one to any feature, and have the feature bar adopt that Index's color. Available Indexes are shown in a strip in the top section of the view.

---

## 1. Data model

### New: `indexes` collection
Add a top-level `indexes` array to app state (persisted alongside features in localStorage).

```
Index = {
  id: string,        // uuid
  name: string,      // "Payments", "Q3 Bets", "Tech Debt"…
  color: string      // hex, chosen from the design-system palette
}
```

### Feature change
Add one nullable field to each feature:

```
feature.indexId: string | null   // references Index.id; null = no index
```

- Default for new + existing features: `null` (no index assigned).
- On load, migrate old saved data: any feature without `indexId` gets `null`.

### Persistence
- Extend the save payload to include `indexes` and each feature's `indexId`.
- Bump the stored-schema version (or defensively default missing keys) so existing saved roadmaps load without error.

---

## 2. Bar color resolution

Today bar color derives from `status`. Introduce a single resolver used everywhere a bar is painted:

```
barColor(feature):
  if feature.indexId and index exists → return that index's color
  else → fall back to existing status color
```

- Route **all** bar rendering (bar fill, and any legend/mini-preview) through `barColor()` so Quarter/Month/Sprint views stay consistent.
- Keep status color as the fallback — an unindexed feature looks exactly as it does today.
- Ensure sufficient text contrast on the bar label: compute luminance of the resolved color and pick dark/light label text automatically (the palette already spans light→dark).

---

## 3. Index bar (top section)

A horizontal strip of Index "chips" in the header area, above the swimlanes.

Each chip:
- Color swatch + name.
- Click a chip → (optional, phase 2) filter/highlight features on that index.
- Hover → edit/delete affordances.

Controls:
- **"+ Index"** button at the end of the strip → opens the create-index popover.
- Empty state: a muted "No indexes yet — add one to color-code features."

Layout: `display:flex; flex-wrap:wrap; gap` so chips reflow; never inline-spaced (survives edits).

---

## 4. Create / edit Index popover

Reuse the existing floating-popover pattern (same one the bar edit control uses).

Fields:
- **Name** — text input.
- **Color** — swatch picker limited to the design-system palette (curated set, not a free color wheel). Show ~6–10 swatches.
- **Save** / **Delete** (delete only in edit mode).

On delete: any feature pointing at that index resets `indexId = null` (bars fall back to status color). Confirm if ≥1 feature uses it.

---

## 5. Assigning an Index to a feature

Add an **Index** selector to the existing feature edit popover (the one with name/status/delete):
- A row of the defined index swatches + a "None" option.
- Selecting one sets `feature.indexId`; the bar recolors immediately.
- If no indexes exist yet, show a small "Create an index first" link that opens the index popover.

---

## 6. Rendering / reactivity checklist
- [ ] `barColor()` used in all three views.
- [ ] Index strip re-renders when indexes change.
- [ ] Feature bar recolors instantly on assign/unassign.
- [ ] Deleting an index unassigns its features and repaints them.
- [ ] Label text contrast auto-adjusts.
- [ ] Export (PNG/current export path) reflects index colors.

---

## 7. Build order (suggested)
1. State + persistence (`indexes`, `feature.indexId`, migration).
2. `barColor()` resolver → wire into all bar rendering.
3. Index strip UI (read-only display first).
4. Create/edit/delete index popover.
5. Index selector in the feature edit popover.
6. Delete-cascade + contrast + export polish.

---

## 8. Open questions for you
1. **One index per feature, or many?** One
2. **Clicking an index chip** — should it filter/dim the board to just that index, or is the strip display-only for now? yes
3. **Color source** — restrict to the design-system palette (recommended), or allow any color? color from the monochrome list within settings
4. **Relationship to `status` color** — index fully overrides status when assigned (plan's assumption), or should status still show somewhere (e.g. a thin edge stripe)? fully override.
