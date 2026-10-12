# CrazyCart.in Design Studio — Bug Fix Implementation Plan (companion to redesign-plan.md)

Scope: fix the **confirmed, reproduced** editor rendering defect in the existing, committed app at `d:\CRAZYCART.IN` (Windows / PowerShell, flat project at root — NO nested folder), then land the remaining studio-UI + homepage/garment redesign (P7–P8) on top of a working editor. Do **not** re-scaffold, do **not** change the locked stack (React 18 + TS + Vite + Tailwind + Framer Motion + Fabric.js v6 + React Router v6 + idb + lucide-react, npm). Every path below is absolute. Run all commands in PowerShell from `d:\CRAZYCART.IN`.

Source-of-truth docs already read: `d:\CRAZYCART.IN\.agents\tasks\design.md` (architecture, data model, geometry contract — CORRECT, must be preserved) and `d:\CRAZYCART.IN\.agents\tasks\redesign-plan.md` (16-item redesign plan — the basis for P8). This file is the authoritative plan for the loop that follows; where it concerns P1–P6 it supersedes everything, and for P7–P8 it defers to redesign-plan.md's item detail.

---

## 1. ROOT CAUSE (traced, not assumed)

The uploaded image **is** added to Fabric's in-memory object model and **is** committed to the authoritative store — that is why it shows correctly in Review and the downloaded mockup. The editor's **live on-screen canvas** is broken by the Fabric canvas lifecycle, not by the artwork pipeline.

### Evidence (exact files/functions)

1. **`src/main.tsx`** (the `<StrictMode>` wrapper around `<App />`). In React 18 **dev** (`vite` on `localhost:5173`, which is how the bug is reproduced), StrictMode intentionally double-invokes effects: **mount → cleanup → mount**.

2. **`src/components/editor/CanvasStage.tsx`** — the engine-creation `useEffect` with an empty dependency array (`[]`). On mount it calls `createEngine(canvasEl, …)`; its cleanup calls `engine.dispose()` and nulls the local `engineRef`. Under StrictMode this runs twice against the **same** `<canvas ref={canvasRef}>` DOM element: engine **A** is created, disposed, then engine **B** is created on that same element.

3. **`src/lib/canvas/CanvasEngine.ts` → `createEngine` / `new Canvas(canvasEl, …)`.** Fabric v6 wraps the `<canvas>` on construction — it inserts a `.canvas-container`, keeps the original as `.lower-canvas`, and creates a sibling `.upper-canvas` (the interaction/selection layer) on top. Disposing and re-constructing a Fabric canvas **over the same element** leaves the live DOM in a corrupted state: a stale `.upper-canvas` from the disposed instance remains on top and swallows pointer events, and the canvas the user actually sees is no longer the render target of the live engine (engine B). Fabric's `dispose()` in v6 is also asynchronous, so the synchronous cleanup → immediate re-create races the teardown. This is a well-documented Fabric-in-React failure mode: objects become unselectable after dispose+recreate over the same element ([Stack Overflow](https://stackoverflow.com/questions/36357627/objects-arent-selectable-after-disposing-and-creating-fabric-canvas-over-the-sa)), and it is triggered specifically by React dev mode running the effect twice ([fabric.js discussion #10321](https://github.com/fabricjs/fabric.js/discussions/10321)). Fabric v6 marks the element with `data-fabric` to prevent double-init ([fabric.js docs/error reference](https://errors.standardbeagle.com/fabricjs/fabric.js/trying-to-initialize-a-canvas-that-has-already-bee/)). _Content was rephrased for compliance with licensing restrictions._

   Net observable effect, matching the report exactly: **invisible** (the visible canvas is detached from engine B's render) and **not selectable / no handles** (the stale upper-canvas eats clicks).

4. **`src/lib/canvas/CanvasEngine.ts` → `addImage`.** It runs `canvas.add(img); canvas.setActiveObject(img); canvas.requestRenderAll(); emitSelection(); emitChange()`. So the image object genuinely lives in the engine's object model and is pushed to the store via `onChange`. The object model is correct; only the on-screen presentation is broken.

5. **`src/lib/canvas/CanvasEngine.ts` → `toArtworkPNG` / `toMockupPNG`.** These rasterize with `canvas.toDataURL({ multiplier, … })`. In Fabric v6 `toDataURL` renders a **fresh offscreen canvas from the object model** (`toCanvasElement`), independent of the on-screen lower/upper canvas DOM. So export reflects the model regardless of the broken live display.

6. **`src/pages/Review.tsx`** never mounts a live Fabric canvas. It loads the saved **thumbnail PNG** (`getThumbnail('thumb:' + state.draftId)`) that `src/pages/Studio.tsx → doSave` produced via `engine.toMockupPNG(...)`. So Review shows a model-derived PNG and always looks correct.

### Why Review/export works while the live editor fails
Review/export read the **in-memory object model** through `toDataURL`/`toCanvasElement` (a fresh render). The live editor relies on Fabric's **on-screen render + upper-canvas hit-testing**, which the dispose→recreate-on-same-element lifecycle corrupts. Same object, two render paths; only the on-screen one is broken.

### Contributing/secondary defects the fix must also close
- **Async-attach to a stale/disposed canvas.** `addImage` / `addSticker` / `loadSide` are async (`await FabricImage.fromURL`). A late load that resolves after a dispose or a side reload attaches to a dead canvas. Needs a generation/`isCurrent` guard that drops stale completions.
- **Backing-store width at mount.** `CanvasStage` seeds width from `stage.clientWidth || 480`, which can be `0` before layout; combined with the lifecycle churn this can briefly render into a mis-sized canvas. Initialization must wait for a measured, non-zero width and re-derive geometry on resize (the engine already rescales in `setCanvasPxWidth`).
- **Two presentations of one model.** The live editor derives from the engine object model while Review derives from a *saved* thumbnail that can be stale. The fix routes both through one authoritative design document + one render path (see §2).

> Note on geometry: the stored contract is self-consistent and must be preserved — **layers** store center `(x,y)` in template units (Fabric CENTER origin), **print areas** store top-left `(x,y)` (see `src/data/products.ts` `AREA_*` and `CanvasEngine.centerOf`/`toArtworkPNG`, `src/lib/canvas/geometry.ts`). Do **not** "fix" this; the acute bug is the lifecycle, not the coordinate math.

---

## 2. THE FIX — one authoritative design document + one render pipeline

**Authoritative design document.** The `StudioStore` (`src/context/StudioProvider.tsx`) `state` is the single source of truth: `productId, colorId, sizeId, materialId, activeSide, quantity, layersBySide` (per-side `DesignLayer[]` with stable `id` from `createId()` — already the case), `selectedLayerId`, `zoom`, `showPrintArea`, plus undo/redo stacks and `loadToken`. Each `DesignLayer` already carries id, side, blobKey/`sticker:<id>` ref, center `(x,y)`, width/height, rotation, opacity, keepAspect, locked, visible, z. There must be **no parallel copies** for toolbar/layers-panel/properties/review — every panel reads this store, and Review re-derives its mockup from it (not from a possibly-stale thumbnail).

**One pipeline.** Structural changes flow **store → engine** (`loadSide`), live interactions flow **engine → store** (`onChange` commit, debounced on pointer-up for undo/redo). The engine is the only module importing `fabric`. The live canvas and the export/review mockup both derive from the same engine object model, which is kept in lockstep with the store.

**Lifecycle fix (the acute bug).** In `CanvasStage.tsx`: create the Fabric engine **exactly once per real mount**, and guarantee a Fabric canvas is never constructed over an element that a previous instance still owns. Concretely: give each engine a freshly created `<canvas>` element (create it in the effect and append it into the stage, remove it on cleanup) **or** fully unwrap on dispose (await Fabric's async `dispose()`, remove the Fabric-added `.upper-canvas`/`.canvas-container`, and clear the `data-fabric` marker) before any re-create; and protect the init against StrictMode's double-invoke so there is never a live stale upper-canvas. Add a generation guard so async `addImage`/`loadSide` completions that resolve after dispose are ignored.

This is one tightly interdependent change (P1–P6). It is implemented as **FEAT-001**; the redesign (P7–P8) is **FEAT-002**.

---

## Verify conventions (every item's Verify line uses these)
- `npx tsc --noEmit` → zero TypeScript errors.
- `npm run build` → `tsc -b && vite build` succeeds (Fabric stays code-split off the marketing chunk).
- `npm run lint` → `eslint . --max-warnings 0` → zero warnings.
- `npx vitest run <path>` → unit/component tests pass; `npm run test` runs the whole suite.
- "Bounded dev smoke" = start `npm run dev` (background or with a timeout), hit `http://localhost:5173`, verify, then STOP it — never leave it blocking.

## Hard invariants (hold at every item)
- Preserve the geometry contract in `src/lib/canvas/CanvasEngine.ts` (Fabric CENTER origin; layer `(x,y)` = box center in template units; print-area `(x,y)` = top-left; rotated-AABB overflow; `sticker:<id>` never hitting IndexedDB; `EXPORT_LONGEST_EDGE=2000`; Textbox normalize-on-commit). Keep per-silhouette viewBoxes and print areas in `src/data/products.ts` unchanged. Keep the already-green carry-over fixes (MED-1 Textbox resize, MED-2 engine tests, NIT-1 EXPORT_LONGEST_EDGE).
- FRONTEND ONLY. WhatsApp handoff only (`wa.me/918208818421`), manual-attach copy, no false "order placed / sent / auto-attached" claims. Base price ₹450 single-sourced in `src/data/pricing.ts`.
- `CREAZYCARTLOGO.png` stays unmodified at root. No nested `crazycart/` folder. No console errors, no broken images.

---

# Implementation Plan (ordered P1 → P8)

## PRIORITY 1 — Fix the image-render pipeline (the acute bug)

- [ ] 1. Fix the Fabric engine lifecycle so the live canvas the user sees/clicks is always the live engine's canvas (root-cause fix).
      In `src/components/editor/CanvasStage.tsx`, stop disposing+recreating Fabric over the same persistent `<canvas>`. Create the Fabric canvas element imperatively inside the init effect (append a fresh `<canvas>` into `stageRef`, pass it to `createEngine`, and on cleanup `await engine.dispose()` then remove that element), and guard the init so StrictMode's mount→cleanup→mount never leaves a stale `.upper-canvas`/`.canvas-container` or a half-disposed instance. Keep the `ResizeObserver`; only initialize once `stageRef` reports a non-zero width (fall back to a `requestAnimationFrame` re-measure instead of a magic `480`).
      Files: `src/components/editor/CanvasStage.tsx`, and if dispose needs to be awaitable, `src/lib/canvas/CanvasEngine.ts` (`dispose` returns the Fabric `dispose()` promise) + the `CanvasEngine.dispose` type in `src/types/editor.ts`.
      Verify: bounded dev smoke on `/studio?product=classic-tshirt` — upload a transparent PNG; it appears on the shirt immediately, shows selection handles, and is draggable; no duplicate/stale canvas in the DOM (one `.canvas-container`, one `.upper-canvas`); no console errors. `npx tsc --noEmit` clean; `npm run lint` zero warnings.

- [ ] 2. Add a generation/`isCurrent` guard so async image loads can't attach to a disposed/re-initialized canvas or clobber newer edits.
      In `src/lib/canvas/CanvasEngine.ts`, capture a monotonically increasing token at the start of each async op (`addImage`, `addSticker`, `loadSide`) and, after each `await FabricImage.fromURL(...)`, bail (skip `canvas.add`) if the engine was disposed or the token is stale. `dispose()` invalidates the token. This closes the "late callback overwrites newer design changes / attaches to a stale canvas" race.
      Files: `src/lib/canvas/CanvasEngine.ts`.
      Verify: new engine unit test (see item 5) covering a dispose-mid-load drops the late add without throwing; `npx vitest run src/lib/canvas` passes.

- [ ] 3. Make the add-image flow render-immediate and store-authoritative end to end (validate → decode → layer → store → live render → auto-select → handles → panels → undo/redo).
      Confirm/repair the chain `Studio.handleAddImage` → `putBlob` → `engine.addImage(opts, resolveLayerSource)` → `canvas.add/setActiveObject/requestRenderAll` → `onChange` → `COMMIT_LAYERS`, so a valid upload becomes a selected, visible layer fitted inside the print area in one step with no user action (no layer-list click, zoom change, side switch, or refresh). Preserve transparent-PNG backgrounds (no background fill on add/export). Ensure a failed decode/oversize surfaces the existing toast and does not add a layer.
      Files: `src/pages/Studio.tsx` (`handleAddImage`), `src/lib/canvas/CanvasEngine.ts` (`addImage`), `src/utils/imageProcessing.ts` / `src/utils/validation.ts` (confirm only).
      Verify: bounded dev smoke — upload PNG/JPG/WebP incl. transparent PNG: layer appears selected with handles, listed in Layers, editable in Properties, and Ctrl+Z removes it / Ctrl+Shift+Z restores it. `npx tsc --noEmit` clean.

- [ ] 4. Route Review/export through the authoritative document so the editor and review share one pipeline (no stale thumbnail divergence).
      Make `src/pages/Review.tsx` derive its mockup from the authoritative `state.layersBySide` + product/color via the shared render path (regenerate through a headless `CanvasEngine` + `toMockupPNG`, or reuse the engine already holding the active side), rather than relying solely on the saved `thumb:` PNG. Keep the saved thumbnail only as a fast first paint / fallback. This guarantees Review always matches the live editor.
      Files: `src/pages/Review.tsx`, and a small shared helper under `src/lib/canvas/` or `src/services/designExport.ts` to build a mockup from serialized layers (reusing `recoloredGarmentMarkup` + `resolveLayerSource`).
      Verify: bounded dev smoke — add artwork, go to `/studio/review`: the preview matches the editor placement exactly; download produces the same composition; `npx tsc --noEmit` clean; `npm run lint` zero warnings.

- [ ] 5. Add the engine-level regression test: an added image layer is actually present and visible on the LIVE canvas with non-zero bounds inside the stage (not only in serialized state).
      Extend `src/lib/canvas/CanvasEngine.test.ts` (using `src/test/fabricTestEnv.ts`): after `engine.addImage(...)`, assert the Fabric canvas has exactly one object; that it is `visible`, `opacity > 0`, `selectable`; that its on-canvas scaled bounds are finite and `> 0` (no NaN/zero/negative) via `getScaledWidth()/getScaledHeight()`; and that its center lies within the stage pixel box `[0,canvasPxWidth] × [0, canvasPxWidth*viewBox.height/viewBox.width]` and within the print area. Add the dispose-mid-load case from item 2. Expose a tiny read-only test accessor on the engine (e.g. `__debugObjectCount()`/`__debugActiveBounds()`) or assert through `serializeSide()` + a live-bounds getter — but the assertion MUST reflect the live canvas object, not just serialized JSON.
      Files: `src/lib/canvas/CanvasEngine.test.ts`, minimal test-only accessor in `src/lib/canvas/CanvasEngine.ts` + `src/types/editor.ts` if needed.
      Verify: `npx vitest run src/lib/canvas` — all prior tests plus the new "live canvas has the added image with non-zero in-stage bounds" and "dispose-mid-load is dropped" tests pass.

## PRIORITY 2 — Canvas geometry & coordinate mapping

- [ ] 6. Harden the single coordinate model across DPR, zoom, pan, and resize without changing stored geometry.
      Audit `CanvasEngine` so template-viewBox units stay canonical and conversions happen only at render/interaction boundaries: `enableRetinaScaling` backing-store scaling must not alter logical coordinates; `setZoom` must change the viewport only (never mutate stored `width/height/x/y`); `setCanvasPxWidth` must preserve logical geometry on resize (image/sticker rescale by factor; Textbox folds scale into width/fontSize — already implemented, keep). Confirm pointer positions are never stored in browser coords and later read as garment coords. Verify desktop↔mobile and zoom changes don't move artwork relative to the garment.
      Files: `src/lib/canvas/CanvasEngine.ts`, `src/components/editor/CanvasStage.tsx` (ResizeObserver → `setCanvasPxWidth`).
      Verify: `npx vitest run src/lib/canvas` (existing resize round-trip + a new zoom-invariance assertion: serialized geometry unchanged after `setZoom` in/out) passes; bounded dev smoke — zoom in/out and resize the window; artwork stays locked to the same spot on the garment.

- [ ] 7. Confirm the printable-area guide maps to the garment preview (not hardcoded page coords) for every product/side.
      Verify `CanvasStage` draws the guide from `getPrintArea(product, side)` as viewBox-relative percentages over the same stage box the garment SVG fills (it does; keep it), and that new artwork is centered in that area via `CanvasEngine.centerOf`/`fitScale`. Confirm overflow uses the rotated-AABB `isOutOfBounds` and the non-blocking warning in `Studio.tsx`.
      Files: `src/components/editor/CanvasStage.tsx`, `src/data/products.ts` (confirm), `src/lib/canvas/geometry.ts` (confirm).
      Verify: bounded dev smoke across tee/cap/tote/mug/sleeve sides — guide aligns to the garment, new art lands inside it, dragging outside raises the warning; `npx tsc --noEmit` clean.

## PRIORITY 3 — Make every transformation work

- [ ] 8. Verify and repair all layer transformations against the live (now-fixed) canvas.
      With the lifecycle fixed, exercise drag, corner-resize (aspect-locked by default; free when `keepAspect` off), rotate about the object center, opacity, duplicate, delete, lock/unlock, visibility, z-order up/down, and undo/redo — all operating on the **currently selected** layer via `CanvasEngine.updateSelected`/`duplicate`/`remove`/`setLock`/`setVisible`/`reorder` and committing to the store on pointer-up. Guard against NaN/Infinity/negative/zero dimensions with sensible min/max; ensure rotation never resets position and the Properties panel shows live rotation/opacity.
      Files: `src/lib/canvas/CanvasEngine.ts`, `src/components/editor/PropertiesPanel.tsx`, `src/components/editor/DesignToolbar.tsx`, `src/pages/Studio.tsx` (selected-layer ops), `src/hooks/useUndoRedo.ts`.
      Verify: bounded dev smoke runs every transform with no jumps/offset/disappearance; locked layers don't move; `npx vitest run src/lib/canvas src/context` passes; `npx tsc --noEmit` clean.

## PRIORITY 4 — Layer selection ↔ canvas two-way sync

- [ ] 9. Make the Layers panel and canvas selection stay in sync both directions.
      Selecting in `LayersPanel` → `engine.selectById` reveals handles and populates Properties with the real values; selecting on canvas → `onSelection` highlights the matching row and updates Properties; `SET_SELECTED` keeps `selectedLayerId` authoritative so a state change never silently switches the active layer. Confirm hidden/locked rows behave (locked not draggable, hidden not rendered but still listed).
      Files: `src/components/editor/LayersPanel.tsx`, `src/components/editor/PropertiesPanel.tsx`, `src/pages/Studio.tsx` (`handleSelection`), `src/lib/canvas/CanvasEngine.ts` (`emitSelection`/`selectById`).
      Verify: bounded dev smoke — two-way selection works, Properties always reflects the active layer, visibility/lock/reorder/duplicate/delete all stay consistent between panel and canvas; `npx tsc --noEmit` clean.

## PRIORITY 5 — Front/back/review consistency

- [ ] 10. Verify per-side preservation and review consistency through the shared document.
      Switching sides serializes the current side into `layersBySide`, swaps garment+print-area, and `loadSide`s the target side (keep the `suppressRef` guard so the reload doesn't echo a spurious commit). Each side's artwork survives flipping and navigation to `/studio/review` and back. Review reflects the authoritative document (item 4).
      Files: `src/pages/Studio.tsx` (side-load effect), `src/lib/canvas/CanvasEngine.ts` (`setSide`/`loadSide`), `src/pages/Review.tsx`.
      Verify: bounded dev smoke — art on front + back, switch sides repeatedly, go to Review and back: nothing is lost or moved; `npx vitest run src/lib/canvas` passes.

## PRIORITY 6 — Export

- [ ] 11. Verify export (artwork-only transparent PNG + labeled mockup) at `EXPORT_LONGEST_EDGE=2000` and WYSIWYG with on-screen placement.
      Confirm `toArtworkPNG` crops to the print area and preserves transparency; `toMockupPNG` composites recolored garment + artwork; both use the fixed export scale, never the on-screen zoom; editor controls/handles never appear in output.
      Files: `src/lib/canvas/CanvasEngine.ts`, `src/services/designExport.ts`.
      Verify: bounded dev smoke — download mockup + artwork; placement matches the editor; transparent background preserved; `npx tsc --noEmit` clean; `npm run build` succeeds.

## PRIORITY 7 — Studio UI polish (redesign)

- [ ] 12. Re-theme the studio UI to the new design system WITHOUT touching the geometry contract (redesign-plan.md item 10, with reliability re-verify from item 14).
      Apply the new tokens to the 3-panel desktop / mobile bottom-sheet layout, keep the canvas large with `touch-action:none`/`overscroll-behavior:contain`, update the print-area guide color to a new-palette token. Do NOT change `CanvasEngine.ts` logic or `products.ts` geometry.
      Files: `src/components/editor/{StudioLayout,CanvasStage,ProductConfigPanel,PropertiesPanel,LayersPanel,DesignToolbar,MobileToolbarSheet,StickerPicker,DesignSummary}.tsx`, `src/pages/Studio.tsx`, `src/components/editor/StudioFallback.tsx`.
      Verify: `npx vitest run src/lib/canvas` still green (contract intact); bounded dev smoke confirms new theme + the P1 image fix still holds; `npx tsc --noEmit` clean; `npm run lint` zero warnings.

## PRIORITY 8 — Homepage + realistic garment SVGs + remaining redesign

- [ ] 13. Complete the remaining redesign per redesign-plan.md (its items 2–9, 11–13, 15–16): tokens/primitives/layout chrome, enriched 17 garment SVGs, hero, home sections, shop, product detail, remaining pages, contact+location, animation/a11y, legacy-alias removal, README.
      This is UI/visual work layered on the fixed editor; follow redesign-plan.md item-by-item (each item there has its own Files/Verify). Preserve the geometry contract and all hard invariants. Items in redesign-plan.md that re-verify studio reliability (its items 1 and 14) are already satisfied by P1–P6 above — just confirm no regression.
      Files: as enumerated in `d:\CRAZYCART.IN\.agents\tasks\redesign-plan.md` items 2–9, 11–13, 15–16.
      Verify: run each redesign-plan.md item's Verify line; then the full gate — `npx tsc --noEmit` clean, `npm run lint` zero warnings, `npm run build` succeeds (Fabric in a separate chunk), `npm run test` all suites pass; bounded dev smoke Home→Shop→Product→Studio(full upload+transform+side-switch+undo+zoom+save+review)→Review→WhatsApp URL with zero console errors; no purple anywhere, no broken images, no fabricated stats.

---

## Decomposition note
P1–P6 (items 1–11) are one tightly interdependent change to the shared design document + render pipeline → **FEAT-001** (`.agents/tasks/task-editor-render-pipeline/features/FEAT-001.json`). P7–P8 (items 12–13) are the separable redesign/UI work → **FEAT-002**. The workflow tail has been restructured into: FEAT-001 coder → FEAT-002 coder → convergence loop (integration fixer + semantic reviewer, verdict-gated) → merge/finalize. If that restructure did not apply, this file is the authoritative plan for the existing implement-and-review loop, run top to bottom.
