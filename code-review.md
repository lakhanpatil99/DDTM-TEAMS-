# CrazyCart.in frontend — full implementation review

The commit lands the entire frontend-only app in one shot: a flat Vite + React 18 + TS SPA with design tokens, a data-driven 8-product catalog, 17 fillable garment SVGs, the Fabric.js studio behind a `CanvasEngine` abstraction, IndexedDB + localStorage drafts, a WhatsApp handoff, and the full marketing/route surface. It faithfully implements the approved design: the hard non-negotiables all hold — no backend/auth/payment anywhere, WhatsApp targets `wa.me/918208818421` built from real selections with explicit manual-attach copy and no false "order received" claims, ₹450 is single-sourced in `data/pricing.ts` and surfaced consistently with no price filter or price sort, the project is flat at the workspace root with `CREAZYCARTLOGO.png` present and unmodified and no nested `crazycart/` folder. The studio geometry contract is largely correct: center origin everywhere, rotated-AABB overflow, `sticker:<id>` resolution that never touches IndexedDB, 2000px export scale, and Textbox scale normalization on commit.

The one real correctness defect is in the canvas-resize path (`setCanvasPxWidth`): it double-scales Textbox wrapping width and leaves `scaleX/scaleY ≠ 1` on text objects, which violates the "fontSize/width authoritative, scale normalized away" invariant and corrupts text size/width on serialize/reload after any resize. That path is wired to a `ResizeObserver`, so it triggers on ordinary layout changes, breaking criterion 13 (exact round-trip) for text. The engine-level tests the plan mandated to catch exactly this (serialize→loadSide round-trip + Textbox normalization) were not written — only pure-geometry math is covered.

**Watch for:** text geometry corruption on canvas resize (confirmed), and the missing `CanvasEngine` round-trip/normalization tests the plan required (confirmed).

**Verdict**: NEEDS_CHANGES

## High-level view

The geometry pivot story is sound. Every Fabric object is created and read with center origin, serialized `(x,y)` is the box center in template units, rotation is `obj.angle`, and overflow uses a rotated-AABB computed from the single center-based definition. `loadSide` derives Fabric scale from `width/height` and never stores `scaleX/scaleY`; `normalizeTextScale` folds Textbox corner-scaling into `fontSize`/`width` on `object:modified` before reading geometry back. Save→reopen is exact for the normal edit flow.

The gap is the responsive resize handler. `CanvasStage` observes its container and calls `engine.setCanvasPxWidth` on every size change. For images that handler correctly multiplies `scaleX/scaleY` by the resize factor, but for Textbox it multiplies `scaleX/scaleY` *and also* `width` by the factor, so the wrapping box grows by `factor²` while the font grows by `factor`, and the object is left with `scaleX/scaleY = factor` instead of 1. Any later serialize misreads text `fontSize` (`obj.fontSize/s`, ignoring the leftover `scaleY`) and `width` (over by the resize factor), and `normalizeTextScale` compounds the width further on the next edit. Text no longer round-trips once the stage has been resized.

The sticker contract is correct end to end: `addSticker` always writes `sticker:<id>`, and `resolveLayerSource` strips that prefix and loads bundled markup through `stickerToDataUrl` (applying `fill` to `[data-recolor]` targets before rasterizing), never reaching IndexedDB, so bundled stickers can't degrade to the missing-blob placeholder. The export path pins 2000px and composites garment-under-artwork with a manual draw order; both export entry points are labeled "preview"/"placement only" and never "print-ready".

Pricing, WhatsApp, and persistence match the spec. ₹450 flows from `PRICING.baseDesignPrice` through `unitPrice`/`totalPrice`; the Stepper clamps `[1,999]` and reverts non-numeric input on blur. The WhatsApp builder emits the locked template with the Design-reference line and the manual-attach line, caps each text layer at 120 chars, and collapses to counts past the ~1500-char soft limit. Save shows "Design saved" only after `saveDraft` resolves `ok`, quota failures surface an error and keep the draft dirty, and auto-save is the same success-gated path on a debounce. The one single-source drift is cosmetic: the export longest-edge is declared as `EXPORT_LONGEST_EDGE = 2000` in `designExport.ts` but the engine hardcodes the literal `2000` in three places instead of importing it.

<details>
<summary>Issues (3)</summary>

1. **Textbox resize corrupts geometry** — `setCanvasPxWidth` scales Textbox `width` by the resize factor on top of scaling `scaleX/scaleY`, and leaves `scaleX/scaleY ≠ 1`; after any `ResizeObserver`-driven resize, text `fontSize`/`width` serialize and reload wrong, breaking exact round-trip (criterion 13). For Textbox, scale either `width`+`fontSize` (keeping scale at 1) or only `scaleX/scaleY` — not both — and re-normalize so no scale survives.
2. **Missing CanvasEngine tests** — plan item 13 required `src/lib/canvas` tests for a serialize→loadSide round-trip of a rotated layer and Textbox scale normalization; only `geometry.test.ts` (pure math) exists. Add the engine round-trip + normalization tests, which also cover the resize regression above.
3. **Export edge not single-sourced** — `CanvasEngine` hardcodes `2000` in `toArtworkPNG`/`toMockupPNG` instead of importing `EXPORT_LONGEST_EDGE` from `designExport.ts`. Import the shared constant so the pinned resolution has one source.

</details>

<details>
<summary>Details</summary>

### Textbox double-scaling in the resize path

`CanvasStage` creates the engine with the measured container width and then observes the stage:

```ts
const ro = new ResizeObserver((entries) => {
  for (const entry of entries) {
    const w = entry.contentRect.width;
    if (w > 0) engine.setCanvasPxWidth(Math.round(w));
  }
});
ro.observe(stage);
```

`setCanvasPxWidth` rescales every object by `factor = s_new / s_old`:

```ts
canvas.getObjects().forEach((o) => {
  o.set({
    left: (o.left ?? 0) * factor,
    top: (o.top ?? 0) * factor,
    scaleX: (o.scaleX ?? 1) * factor,
    scaleY: (o.scaleY ?? 1) * factor,
  });
  if (o instanceof Textbox) o.set({ width: (o.width ?? 0) * factor });
  o.setCoords();
});
```

For an image the single `scaleX/scaleY *= factor` is the correct, standard rescale. For a Textbox the extra `width *= factor` makes the rendered wrapping width (`width * scaleX`) grow by `factor²` while the font (`fontSize * scaleY`) grows by `factor`, and the object is left with `scaleX = scaleY = factor` rather than 1. Nothing re-normalizes it here. The next serialize reads text size as `obj.fontSize / s` and width via `getScaledWidth() / s`:

```ts
fontSize: (o.fontSize ?? 32) / s,          // ignores the leftover scaleY
// unitBox: width = o.getScaledWidth() / s  // width*scaleX = width*factor²  →  over by factor
```

So after a resize, text serializes with `fontSize ÷ factor` and `width × factor`, and reloads at the wrong size. `normalizeTextScale` only runs on `object:modified`, and when it does it folds the stale `scaleX` into width again (`width = obj.width * obj.scaleX`), compounding the error across resizes. Because the resize path is driven by a `ResizeObserver` on the stage container, it fires on window resize, orientation change, panel collapse, and mobile keyboard show/hide — not rare events. This is the one place the otherwise-correct "scaleX/scaleY normalized away, width/fontSize authoritative" invariant is broken.

### Test coverage versus the plan

Plan item 13's verify line calls for `npx vitest run src/lib/canvas` covering a serialize→loadSide round-trip of a rotated layer (center-origin AABB stable) and Textbox scale normalization. The shipped suite has `geometry.test.ts` only, which exercises `rotatedAABB`/`isOutOfBounds` in isolation — good coverage of the overflow math, but it never instantiates the engine, so neither the round-trip nor the normalization path is tested. Those are exactly the tests that would have surfaced the resize defect above. The commit's "22 tests passing" evidence is accurate but does not include the engine-level cases the plan required.

### Export resolution single-sourcing

`designExport.ts` declares `export const EXPORT_LONGEST_EDGE = 2000;` per design NIT-1, but the engine recomputes the scale from a bare literal in both export methods:

```ts
const se = 2000 / viewBox.width;   // toArtworkPNG and toMockupPNG
```

Behavior is correct (both exports and the stage stay WYSIWYG at `sₑ = 2000 / viewBox.width`), but the pinned number now lives in two places. Importing the constant keeps the single source the design asked for. Minor.

</details>

<details>
<summary>File map</summary>

Single commit `678be5a` — 209 files, 19,303 insertions. Key files reviewed:

- `src/lib/canvas/CanvasEngine.ts` — Fabric abstraction; center-origin geometry, serialize/load, export, resize (contains the Textbox resize defect).
- `src/lib/canvas/geometry.ts` + `geometry.test.ts` — rotated-AABB overflow math and its only tests.
- `src/lib/canvas/blobResolver.ts` / `stickerSource.ts` — `sticker:<id>` resolution and recolor-before-raster (correct).
- `src/services/designExport.ts` — `EXPORT_LONGEST_EDGE`, labeled previews.
- `src/services/whatsapp.ts` — locked template, 120-char/1500-char caps, `wa.me/918208818421` (correct).
- `src/data/pricing.ts` — ₹450 single source; `src/components/ui/Stepper.tsx` — [1,999] clamp + revert.
- `src/components/products/FilterBar.tsx` + `src/pages/Shop.tsx` — category/color/style filters, featured/name/newest sort, no price facet (correct).
- `src/pages/Studio.tsx` / `src/context/StudioProvider.tsx` — save success-gating, auto-save, undo/redo cap 50.
- `src/components/editor/CanvasStage.tsx` — ResizeObserver → `setCanvasPxWidth` wiring.

Full diff: `git show 678be5a`.

</details>
