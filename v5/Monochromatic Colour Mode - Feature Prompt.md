# Feature Prompt: Monochromatic Colour Mode for Swimlanes & Features

Add a **monochromatic colour mode** to a swimlane/Gantt-style roadmap timeline app, alongside the existing multi-hue palette. Implement it exactly as specified below.

## 1. Settings control
Add a **"Swimlane colours"** row to the Settings modal, styled like the other rows (icon chip + title + description + a segmented control). It has:
- A **mode switch** — a two-way segmented control: **Multicolour** / **Monochrome**.
  - Multicolour = the existing behaviour (a distinct hue per lane). Unchanged.
  - Monochrome = the new behaviour below.
- When Monochrome is selected, reveal a **"Base colour"** picker: a wrapping row of **12 circular swatches**, one per wheel hue, single-select. The selected swatch shows a ring/border; swatches are dimmed (opacity ~0.5) when Multicolour is active.
- Persist both the mode and the selected hue to localStorage; read them on mount. The Base-colour picker only shows when Monochrome is active.

## 2. The 12 base colours (colour wheel, ~30° apart)
Red 0° `#D64545`, Orange 30° `#D97828`, Amber 48° `#C9A227`, Lime 90° `#7FA82B`, Green 120° `#3C9A5F`, Teal 150° `#2C9E86`, Cyan 180° `#2A9AA8`, Azure 205° `#2C77B5`, Blue 232° `#3A5BD9`, Violet 262° `#6A4CC0`, Magenta 290° `#A64CB0`, Rose 330° `#CE4488`.
Store each as `{ name, hue, hex }`. The **hue** value is what drives the generated tints; the hex is only for rendering the swatch.

## 3. Monochrome colour-generation logic
When monochrome mode is on, replace the lane-colour assignment (keep lane ordering untouched). For the ordered list of unique lanes, colour them in **groups of three**:

- `block = Math.floor(laneIndex / 3)` → which base colour this group uses.
- `within = laneIndex % 3` → position inside the group (0,1,2).
- Base hue for the group = `MONO_BASES[(anchorIndex + block) % 12].hue`, where `anchorIndex` is the index of the user-picked base colour in the 12-hue wheel. Each successive group of 3 lanes **advances to the next hue on the wheel**, wrapping with `% 12` past the 12th.
- Three fixed **high-contrast lightness tints** per hue: `[86, 72, 58]` (light / mid / deep).
- **Alternate the tint direction per group (boustrophedon):**
  - Even groups (block 0, 2, 4 …) run **light → mid → deep**: `shade = within`.
  - Odd groups (block 1, 3, 5 …) run **deep → mid → light**: `shade = 2 - within`.
  - So lanes read: `86,72,58, 58,72,86, 86,72,58, …` across successive groups.
- Each lane's colours:
  - `fill  = hsl(hue, 46%, tints[shade]%)`  (bar fill + lane square)
  - `ink   = hsl(hue, 54%, 28%)`            (bar/label text, same hue, dark, for contrast)
  - `soft  = hsl(hue, 44%, 95%)`            (lane caption background, very light tint)

## 4. Scope of recolouring
The generated `fill`/`ink`/`soft` flow through everything that reads lane identity: swimlane header squares, lane caption backgrounds, feature bars (fill + text), and swimlane filter chips. Leave status signals unchanged — trophy chip, navy "completing-this-quarter" outline, orange "now" markers, dashed unresolved-date outline.

## 5. Defaults & edge cases
- Default mode: Multicolour. Default base hue if none saved: Blue (232°).
- Fewer than 3 lanes in the final group just uses the first 1–2 tints of that group's hue.
- More than 36 lanes simply loops the wheel via `% 12`.
- Both light and dark app themes must remain legible; the 86/72/58 tints with `ink` at 28% lightness keep bar text readable across all 12 hues.
