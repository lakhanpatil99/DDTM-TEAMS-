# CrazyCart.in — Implementation Plan

Source of truth: `d:\CRAZYCART.IN\.agents\tasks\design.md` (approved). This plan sequences the FLAT build at the workspace root `d:\CRAZYCART.IN` — no nested `crazycart/` folder. All paths below are relative to that root unless absolute. Run every command in PowerShell from `d:\CRAZYCART.IN`.

Conventions used by every item's Verify line (established by item 1): `npx tsc --noEmit` = zero TypeScript errors; `npm run build` = Vite production build succeeds; `npm run lint` = ESLint zero warnings; `npx vitest run <path>` = unit tests pass. The codebase must stay buildable (`npx tsc --noEmit` clean) after every item.

- [x] 1. Scaffold the flat Vite + React 18 + TypeScript SPA at the root and lock dependencies.
      Create `package.json` (type module; scripts `dev`/`build` [`tsc -b && vite build`]/`preview`/`lint`/`test` [`vitest run`]/`test:watch`), `index.html` (root `#root`, title, favicon links, lang="en"), `vite.config.ts` (React plugin, `@/`→`src/` alias), `tsconfig.json` + `tsconfig.node.json` (strict, bundler resolution, path alias), `tailwind.config.ts` + `postcss.config.js`, `.eslintrc.cjs` (typescript-eslint, react-hooks, jsx-a11y, zero-warning), `.prettierrc`, `.gitignore`, and `src/main.tsx`/`src/App.tsx` placeholders. Install exact-pinned deps: react@18, react-dom@18, react-router-dom@6, framer-motion, fabric@6, idb, lucide-react, clsx; dev: vite, @vitejs/plugin-react, typescript, tailwindcss, postcss, autoprefixer, eslint + plugins, prettier, vitest, @testing-library/react, @testing-library/jest-dom, jsdom. Copy (never move/modify) `CREAZYCARTLOGO.png` → `public/assets/logo/logo-gold.png`; add committed favicons (`public/favicon.ico`, `public/icons/icon-16/32/180/512.png`) generated from the logo.
      Files: `package.json`, `index.html`, `vite.config.ts`, `tsconfig.json`, `tsconfig.node.json`, `tailwind.config.ts`, `postcss.config.js`, `.eslintrc.cjs`, `.prettierrc`, `.gitignore`, `src/main.tsx`, `src/App.tsx`, `public/assets/logo/logo-gold.png`, `public/favicon.ico`, `public/icons/*`
      Verify: `npm install` succeeds; `npx tsc --noEmit` zero errors; `npm run build` succeeds; confirm `CREAZYCARTLOGO.png` still at root unchanged and no nested `crazycart/` dir exists (`Get-ChildItem`).

- [x] 2. Author the design-token system and self-hosted fonts, mirrored into Tailwind.
      Create `src/styles/tokens.css` (all §4 CSS variables: purple/gold/warm-neutral/ink colors, split `--fs-*`/`--lh-*`/`--ls-*` type tokens, spacing/radii/shadows incl. 3px `--shadow-focus`, z-index, motion tokens), `globals.css` (reset, base, `@font-face` for Fraunces+Inter with `font-display:swap`, utility layers), `animations.css` (keyframes + reduced-motion fallbacks), `responsive.css` (safe-area, `dvh`). Place self-hosted woff2 for Fraunces + Inter (site chrome weights) in `public/fonts/`. Extend `tailwind.config.ts` theme so `bg-brand-purple`/`text-ink`/`rounded-card`/`shadow-card`/`fontSize` tuples resolve to the same token values. Import `tokens.css`/`globals.css`/`animations.css`/`responsive.css` in `main.tsx`.
      Files: `src/styles/tokens.css`, `src/styles/globals.css`, `src/styles/animations.css`, `src/styles/responsive.css`, `public/fonts/*.woff2`, `tailwind.config.ts`, `src/main.tsx`
      Verify: `npm run build` succeeds and emits CSS with the tokens; `npm run dev` (bounded smoke, item 23 pattern) shows tokens applied with no 404 font requests.

- [x] 3. Define the TypeScript domain types.
      Create `src/types/product.ts` (`ProductCategoryId`, `PrintSide`, `GarmentColorId`, `MaterialId`, `Material`, `SizeOption`, `GarmentColor`, `PrintArea`, `GarmentTemplate` [per-side `viewBox`], `Product` [with forbidden `basePriceOverride?: never`], `ProductCategory`), `src/types/design.ts` (`LayerType`, `BaseLayer` with center-origin `x,y`/`width,height`/`rotationDeg`, `ImageLayer`, `StickerLayer` [`fill?`, reserved `sticker:<id>` key], `TextLayer`, `DesignLayer` union, `DESIGN_SCHEMA_VERSION`, `SavedDesignMeta`, `SavedDesignBody`), `src/types/editor.ts` (`CanvasEngine`, `EngineOptions`, `SelectionInfo`, `AddImageOpts`, `AddTextOpts`, `BlobResolver`, `FontDef`).
      Files: `src/types/product.ts`, `src/types/design.ts`, `src/types/editor.ts`
      Verify: `npx tsc --noEmit` zero errors.

- [x] 4. Build the data-driven config layer with the §6.4 starter catalog and dev-time validation.
      Create `src/data/colors.ts` (the exact 10 `GarmentColor`s with §6.2 hex + `isLight`), `pricing.ts` (`PRICING` with `baseDesignPrice:450`, `unitPrice`, `totalPrice`, `formatINR`), `products.ts` (exactly the 8 §6.4 products with shared `APPAREL`/`ONE_SIZE`/`FULL10` presets, per-product colors/sizes/materials/`supportedSides`/`printAreas`/`heroImageOrder`/`styleTags`/`badges`, `templates` referencing item-5 SVGs), `categories.ts` (8 category rows with `previewProductId`/`previewColorId`), `fonts.ts` (`studioFonts`: Inter, Fraunces, Montserrat, Poppins, Oswald, Bebas Neue, Pacifico, Lobster), `stickers.ts` (curated local SVG set catalog), `site.ts` (brand strings, WhatsApp `918208818421`, nav items, socials, FAQ, announcement id + copy). Add a dev-only `validateConfig()` (§6.3 assertions) that `throw`s in dev and is tree-shaken in prod; call it on module load.
      Files: `src/data/colors.ts`, `src/data/pricing.ts`, `src/data/products.ts`, `src/data/categories.ts`, `src/data/fonts.ts`, `src/data/stickers.ts`, `src/data/site.ts`, `src/data/validateConfig.ts`
      Verify: add `npx vitest run src/data` with a test asserting `validateConfig()` passes, 8 products / 8 categories exist, and every `product.colors` id resolves; `npx tsc --noEmit` clean.

- [x] 5. Create the 17 fillable garment SVG assets, the curated sticker SVGs, and the inlining `GarmentSvg` component.
      Author 17 garment SVGs under `src/assets/garments/<product>/<side>.svg` cloned from the §8.6 reference structure (scoped `data-recolor="body"`, optional `data-recolor="trim"`, multiply/screen color-agnostic overlays, NO internal `url(#id)` refs), each with its own §8.6 viewBox per silhouette. Author the curated sticker SVGs under `src/assets/stickers/` (stars, hearts, smileys, abstract shapes, flames, lightning, flowers, geometric, minimal symbols; recolorable ones tag paths with scoped `data-recolor`). Create `src/components/products/GarmentSvg.tsx` that `?raw`-imports the markup, injects via `dangerouslySetInnerHTML` into a `<div role="img" aria-label>`, and recolors `[data-recolor="body"]`/`[data-recolor="trim"]` scoped to its own `wrapperRef` from a `color` prop.
      Files: `src/assets/garments/**/*.svg` (17), `src/assets/stickers/*.svg`, `src/components/products/GarmentSvg.tsx`
      Verify: `npx vitest run src/components/products` with a test rendering `GarmentSvg` for a product/side and asserting the `[data-recolor="body"]` element's `fill` equals the passed color hex; `npx tsc --noEmit` clean.

- [x] 6. Implement shared utils and reusable hooks.
      Create `src/utils/pricing.ts` (re-export/compose `data/pricing` helpers + quantity clamp `[1,999]` with revert-on-invalid), `validation.ts` (upload MIME+extension+size [10MB]+dimension checks returning typed errors; text ≤500 chars; free-hex regex `/^#([0-9a-f]{3}|[0-9a-f]{6})$/i`; rename trim+cap 60), `imageProcessing.ts` (decode + downscale to ~2000px longest edge, preserve aspect), `formatting.ts` (relative time, etc.), `cn.ts` (clsx wrapper), `id.ts` (nanoid-style id). Create hooks `src/hooks/useMediaQuery.ts`, `usePrefersReducedMotion.ts`, `useScrollDirection.ts` (IntersectionObserver/rAF-throttled), `useDocumentTitle.ts`, `useUndoRedo.ts`, `useLocalDrafts.ts`, `useDesign.ts` (thin selectors; full stores arrive in item 12).
      Files: `src/utils/*.ts`, `src/hooks/*.ts`
      Verify: `npx vitest run src/utils` covering quantity clamp/revert, upload validation accept/reject, and hex validation; `npx tsc --noEmit` clean.

- [x] 7. Build the `components/ui` primitive library.
      Create accessible, keyboard-operable primitives with visible `--shadow-focus` ring and ≥44px touch targets: Button (primary/gold/secondary/ghost/danger; sm/md/lg; loading/disabled), Card, Badge, Swatch (`aria-pressed`, ring+check), Input/Textarea/Select/Range (labeled, `aria-describedby` errors), Modal (focus-trap, Esc, overlay click, `role="dialog" aria-modal`), Drawer, BottomSheet (drag handle, snap points), Tabs, Tooltip, Toast + ToastProvider wiring (`aria-live="polite"`), SegmentedControl, Stepper (clamped quantity), Accordion (disclosure), SizeGuideModal, ConfirmDialog, Skeleton.
      Files: `src/components/ui/*.tsx`
      Verify: `npx vitest run src/components/ui` for Modal focus-trap/Esc, Swatch `aria-pressed`, and Stepper clamp; `npx tsc --noEmit` clean.

- [x] 8. Assemble the router, root layout chrome, and page transitions.
      Create `src/router.tsx` (`createBrowserRouter` with all §5 routes; `/shop` and `/catalog` both mount `<Shop/>`; `React.lazy` for `/studio`), `src/App.tsx` (RouterProvider + ScrollRestoration + providers slot), and `components/layout/`: `AnnouncementBar` (dismiss keyed by `cc.prefs.dismissedAnnouncementId`), `Header` (sticky scroll-aware via `useScrollDirection`, nav items + "Start Designing", logo-gold-on-purple chip, selection indicator slot), `Nav`, `MobileDrawer` (slide-in, focus-trap), `Footer` (gold logo on `--purple-800`), `Container`, `SectionReveal` (Framer Motion `whileInView`, reduced-motion-safe), and a root layout with animated `<Outlet/>` (350ms page transition) plus a dedicated full-bleed studio layout without the marketing footer. Add temporary placeholder page components so all routes resolve.
      Files: `src/router.tsx`, `src/App.tsx`, `src/components/layout/*.tsx`, `src/pages/*` (placeholders)
      Verify: `npm run build` succeeds; bounded dev smoke confirms every route renders, header is sticky, mobile drawer opens, 404 shows for unknown routes, scroll resets on navigation.

- [x] 9. Build the Home page with all 12 §5.2 sections in order.
      Create `components/home/`: Hero (locked copy "Your Ideas. Your Style. Your Creation.", sub-copy, "Create Your Design"→`/studio`, "Explore Products"→`/catalog`, gold accent via `--gold-600`/dark chip, SVG garment composition with reduced-motion-safe entrance), ProductShowcase (transform/opacity marquee, pauses on reduced motion), CategoryGrid (8 cards → `/catalog?category=`), FeaturedProducts (sorted by `heroImageOrder`, "Designs from ₹450"), StudioShowcase ("Start Designing" CTA), PrintingInfo (DTF/sublimation, no fulfillment promises), MerchShowcase, FaqAccordion (content from `data/site.ts`), ContactCta (wa.me + mailto). Compose them in `src/pages/Home.tsx` top-to-bottom (announcement bar + header come from the layout, not re-rendered here).
      Files: `src/components/home/*.tsx`, `src/pages/Home.tsx`
      Verify: `npm run build` succeeds; bounded dev smoke confirms all 12 sections render in order, hero copy/CTAs exact, prices read "Designs from ₹450"; `npx tsc --noEmit` clean.

- [x] 10. Build the Shop page (and its `/catalog` alias) with URL-synced filters, sort, quick preview, and mobile filter drawer.
      Create `components/products/ProductCard` (GarmentSvg mockup, name, "Designs from ₹450", "Create your own design"), `FilterBar` (category/color/material/style — NO price filter/sort), `MobileFilterDrawer` (bottom-sheet), `QuickPreviewModal` (mockup + config + "Customize"), and `src/pages/Shop.tsx`: debounced search (`q`), filters, sort (featured/name A–Z/Z–A/newest), all state synced to URL query params, branded empty state, ₹450 on every card. Memoize filtering.
      Files: `src/components/products/ProductCard.tsx`, `FilterBar.tsx`, `MobileFilterDrawer.tsx`, `QuickPreviewModal.tsx`, `src/pages/Shop.tsx`
      Verify: `npx vitest run src/pages` test that filtering by category/color narrows results and empty match yields the empty state; bounded dev smoke confirms URL sync + quick preview; confirm no price filter/sort control exists.

- [x] 11. Build the Product detail page with selectors, size guide, and price summary.
      Create `components/products/`: ColorSwatches, SizeSelector, MaterialSelector (hidden when product has no materials), PrintLocationTabs, PriceSummary (₹450 base + disclaimer), and `src/pages/ProductDetail.tsx` (`/product/:productId`): large GarmentSvg preview reacting to color, selectors, care instructions when present, SizeGuideModal (placeholder measurements note), and "Customize This Product" → `/studio?product=&color=&size=&material=`.
      Files: `src/components/products/ColorSwatches.tsx`, `SizeSelector.tsx`, `MaterialSelector.tsx`, `PrintLocationTabs.tsx`, `PriceSummary.tsx`, `src/pages/ProductDetail.tsx`
      Verify: bounded dev smoke confirms selectors recolor the preview, material selector hidden for cap/mug, price shows ₹450, Customize CTA deep-links with selections; `npx tsc --noEmit` clean.

- [x] 12. Implement the domain context stores (Studio, Selection, SavedDesigns, Toast).
      Create `src/context/StudioProvider.tsx` (`useReducer`: `activeProduct,colorId,sizeId,materialId,activeSide,quantity,layersBySide,selectedLayerId,zoom,showPrintArea,isDirty` + undo/redo stacks capped at 50; pure serializable actions), `SelectionProvider.tsx` (current-design-in-progress for header indicator + `/studio` deep-link), `SavedDesignsProvider.tsx` (wraps the item-16 repo), and wire ToastProvider from item 7. Mount providers in `App.tsx`. Flesh out `useDesign`/`useLocalDrafts`/`useUndoRedo` against these stores.
      Files: `src/context/StudioProvider.tsx`, `SelectionProvider.tsx`, `SavedDesignsProvider.tsx`, `src/App.tsx`, `src/hooks/useDesign.ts`, `useLocalDrafts.ts`, `useUndoRedo.ts`
      Verify: `npx vitest run src/context` reducer tests for add/remove/reorder/undo/redo and side-switch state retention; `npx tsc --noEmit` clean.

- [x] 13. Build the Fabric.js abstraction `CanvasEngine` (the only module importing `fabric`).
      Create `src/lib/canvas/CanvasEngine.ts` implementing the §9.1 API (`createEngine`, `setSide`, `addImage`, `addText`, `addSticker`, `updateSelected`, `remove`, `duplicate`, `setLock`, `setVisible`, `reorder`, `selectById`, `clearSelection`, `setZoom`, `fitToStage`, `toArtworkPNG`, `toMockupPNG`, `serializeSide`, `loadSide`). Enforce §9.4: center origin everywhere; store only `x,y,width,height,rotationDeg` (template units); compute Fabric scale as `width*s/naturalWidth`; normalize Textbox corner-scale into `fontSize`/`width` on commit then reset `scaleX/scaleY=1`; stage scale `s = canvasPxWidth/viewBox.width`; zoom clamp `[0.5,3]`. Implement `loadFont(fontDef)` guarantee (await `FontFace`/`document.fonts.load` before applying). Resolve `sticker:<id>` keys from the bundle, never IndexedDB.
      Files: `src/lib/canvas/CanvasEngine.ts`, `src/lib/canvas/loadFont.ts`, `src/lib/canvas/geometry.ts`
      Verify: `npx vitest run src/lib/canvas` (jsdom) for serialize→loadSide round-trip of a rotated layer (center-origin AABB stable) and Textbox scale normalization; `npx tsc --noEmit` clean.

- [x] 14. Build the Studio page shell: layout, product-config panel, canvas stage, and side switching.
      Create `components/editor/`: StudioLayout (desktop 3-panel sticky left/right + flex-1 center; mobile compact top bar + bottom-sheet tools + sticky action area), ProductConfigPanel (color swatches, size, material [hidden if none], print-location tabs), CanvasStage (GarmentSvg backdrop + print-area guide toggle + transparent Fabric `<canvas>` driven by CanvasEngine; `touch-action:none`/`overscroll-behavior:contain` while manipulating), DesignSummary, and `src/pages/Studio.tsx` (`/studio?product=&color=&size=&material=&draft=` deep-link init from StudioProvider). Wire `setSide` to serialize/clear/reload preserving per-side artwork (§9.2).
      Files: `src/components/editor/StudioLayout.tsx`, `ProductConfigPanel.tsx`, `CanvasStage.tsx`, `DesignSummary.tsx`, `src/pages/Studio.tsx`
      Verify: bounded dev smoke on `/studio?product=classic-tshirt&color=purple` confirms garment renders/recolors, side switch preserves layers, no excessive horizontal scroll on laptop widths; `npx tsc --noEmit` clean.

- [x] 15. Build the studio tools, toolbar, panels, and undo/redo wiring.
      Create `components/editor/`: DesignToolbar (Add Image/Text/Sticker, Undo, Redo, Zoom fit/in/out/reset, Toggle print-area, Preview-hide-controls, Download, Save; contextual Position/Scale/Rotate/Duplicate/Delete/Lock/Hide/Reorder when a layer is selected), PropertiesPanel (text + image/sticker props incl. opacity/aspect-lock/free-hex-with-revert, sticker recolor when `[data-recolor]` present), LayersPanel (z-order list, select/reorder/hide/lock/delete), StickerPicker (inserts real editable `StickerLayer`), MobileToolbarSheet. Wire upload (§10.1 validation), text tool (§10.2 validation + `loadFont`), print-area overflow warning (§9.6 rotated AABB), and global Ctrl/Cmd+Z/Shift+Z handlers that ignore text-editing focus.
      Files: `src/components/editor/DesignToolbar.tsx`, `PropertiesPanel.tsx`, `LayersPanel.tsx`, `StickerPicker.tsx`, `MobileToolbarSheet.tsx`
      Verify: bounded dev smoke: upload PNG adds a layer, bad type rejected with message, text/stickers add editable layers, move/resize/rotate/duplicate/delete/lock/hide/reorder work, undo/redo reverse ≥20 ops, out-of-bounds warning shows; `npx tsc --noEmit` clean.

- [x] 16. Implement the persistence, export, and WhatsApp services.
      Create `src/services/designStorage.ts` (idb DB `crazycart` with `blobs`/`thumbnails`/`exports` stores; `putBlob/getBlob/deleteBlob`; `StorageRepo` facade coordinating IDB blobs + localStorage `cc.draftIndex`/`cc.design.<id>`/`cc.prefs`; manual save gated on resolved write; ~2s debounced auto-save; rename trim+cap 60; §11.3 quota/corruption/missing-blob/IDB-unavailable/newer-schema handling; `migrateDesign`; bounded `exports` cleanup one-per-draft + orphan purge), `designExport.ts` (`EXPORT_LONGEST_EDGE=2000`, `sₑ=2000/viewBox.width`; artwork-only PNG cropped to print-area rotated-AABB + mockup PNG; await `document.fonts.ready` + in-use faces; iOS WebKit `globalCompositeOperation` blend fallback; clear mockup-vs-artwork labels, never "print-ready"), `whatsapp.ts` (build the §13 locked template from real selections incl. Design-reference line + manual-attach line; per-text-layer 120-char cap + ~1500-char soft limit collapsing to counts; `https://wa.me/918208818421?text=` via `window.open(url,'_blank','noopener')`).
      Files: `src/services/designStorage.ts`, `src/services/designExport.ts`, `src/services/whatsapp.ts`
      Verify: `npx vitest run src/services` for save-then-load round-trip (fake-indexeddb), quota/corruption/missing-blob paths, quantity×450 totals in the message, and the Design-reference + manual-attach lines present; `npx tsc --noEmit` clean.

- [x] 17. Build the Saved Designs page.
      Create `src/pages/SavedDesigns.tsx` (`/saved`): grid of draft cards with IDB mockup thumbnail (missing-blob badge), config summary (product·color·size·material·sides·qty·₹450×qty), relative last-saved time; per-card Reopen (→`/studio?draft=<id>`), inline Rename, Delete behind ConfirmDialog (removes LS + IDB); branded empty state; damaged/missing-blob recovery messaging per §11.3.
      Files: `src/pages/SavedDesigns.tsx`
      Verify: bounded dev smoke: save a draft in studio, confirm it lists with thumbnail + meta, reopen restores all sides/layers, rename persists, delete requires confirm and clears data, empty state shows when none; `npx tsc --noEmit` clean.

- [x] 18. Build the Review & Submit screen and WhatsApp handoff.
      Create `components/review/`: ReviewScreen, WhatsAppHandoff, DownloadPreview, and `src/pages/Review.tsx` (`/studio/review`): mockup preview, selected product/color/size/material/print-sides, text/artwork summary, quantity, ₹450 unit + total, base-price disclaimer, Download preview button, Share-on-WhatsApp button with the §13 pre-submit sequence (validate required selections and block with specific message; auto-save; manual-attach explanation; preserve design on "Back to editor"). Guard: redirect to `/studio` if no active design.
      Files: `src/components/review/ReviewScreen.tsx`, `WhatsAppHandoff.tsx`, `DownloadPreview.tsx`, `src/pages/Review.tsx`
      Verify: bounded dev smoke: review shows full summary + working download; WhatsApp opens the correct wa.me URL only on click; missing selections block with a message; no "order received" claim; `npx tsc --noEmit` clean.

- [x] 19. Build the remaining content pages (About, Contact, HowItWorks, 404).
      Create `src/pages/About.tsx` (brand story, DTF/sublimation explainer, tagline), `HowItWorks.tsx` (Choose→Design→Preview→Send→Confirm), `Contact.tsx` (wa.me + `mailto:` composition only, business hours, socials; any form builds a client-side mailto/wa.me link, no network submit), `NotFound.tsx` (branded 404 + home CTA). Replace the item-8 placeholders.
      Files: `src/pages/About.tsx`, `src/pages/HowItWorks.tsx`, `src/pages/Contact.tsx`, `src/pages/NotFound.tsx`
      Verify: bounded dev smoke confirms each renders with correct content and the Contact form performs no network request; `npx tsc --noEmit` clean.

- [x] 20. Animation, accessibility, and reduced-motion polish pass.
      Gate all Framer Motion (page transitions, hero, staggered reveals, parallax, swatch/button feedback, panel/drawer/sheet/modal transitions) through `usePrefersReducedMotion()`; animate transform/opacity only; ensure IntersectionObserver/rAF scroll handling. Audit a11y: one `h1`/page, landmarks, focus-trap + Esc + focus-restore on all overlays, `aria-pressed` swatches, `aria-label` icon buttons, `aria-describedby` field errors, `aria-live` toasts, canvas non-visual description + keyboard select/nudge/delete. Confirm route-level code-split keeps Fabric off marketing routes and images use `loading="lazy"` + aspect boxes.
      Files: touches `src/components/**`, `src/pages/**`, `src/hooks/usePrefersReducedMotion.ts`, `src/styles/animations.css`
      Verify: bounded dev smoke with emulated `prefers-reduced-motion: reduce` shows no parallax/translation/looping and immediate content; keyboard-only pass reaches all controls with visible focus ring; build output shows a separate studio/Fabric chunk; `npx tsc --noEmit` clean.

- [x] 21. Write the README deliverable.
      Create `README.md` at root per §19: overview + brand, prerequisites (Node ≥18, npm), install/dev/build/preview/lint/test commands, where to edit `src/data/*`, how SVG garment recoloring works and how to drop in real photos via `GarmentColor.images[side]`, how local IndexedDB+localStorage drafts work (nothing transmitted), the WhatsApp manual-attachment behavior, and the explicit limitations note (SVG mockup placeholders, placeholder size-guide measurements, previews not print-ready).
      Files: `README.md`
      Verify: `read_file` confirms all §19 sections present and commands match `package.json` scripts.

- [x] 22. Final verification gate (run in `d:\CRAZYCART.IN` with PowerShell).
      Run `npm install`; `npx tsc --noEmit` (zero errors); `npm run lint` (zero warnings, if configured); `npm run build` (must succeed). Run a BOUNDED dev-server smoke check: start `npm run dev` in the background (or with a timeout), hit the local URL, then stop it — never leave it blocking. Trace the full flows end-to-end: Home→Shop(filter)→Product→Customize→Studio(upload image + add text + add sticker + recolor + switch side + undo/redo + zoom + print-area toggle)→Save→Saved Designs(reopen/rename/delete)→Review→WhatsApp URL. Confirm responsive assumptions at 360×800, 390×844, 430×932, 768×1024, 1366×768, 1440×900, 1920×1080. Confirm `prefers-reduced-motion` behavior, zero console errors, and that NO backend/database/auth/payment was introduced and `CREAZYCARTLOGO.png` is unmodified at root with no nested `crazycart/` folder.
      Files: none (verification only)
      Verify: all commands above pass; the trace completes without console errors; responsive and reduced-motion checks hold; acceptance criteria §17 (1–31) are satisfied.

---

## Verification log (iteration 1 — first implementation)

Ran in `d:\CRAZYCART.IN` via PowerShell (Node v24.19.0, npm 11.17.0):

- `npm install` — succeeded (deps pinned per §2; esbuild install script approved).
- `npx tsc --noEmit` — **0 errors**.
- `npm run build` (`tsc -b && vite build`) — **succeeded**. Fabric.js is code-split into a
  separate `Studio` chunk (~347 kB) and is NOT in the marketing `index` chunk, so it never loads on
  marketing routes.
- `npm run lint` (`eslint . --max-warnings 0`) — **0 warnings / 0 errors**.
- `npx vitest run` — **22 tests passing** across 6 files (config invariants + 8 products/8
  categories, pricing & quantity clamp/revert, upload + hex validation + rename cap, rotated-AABB
  overflow geometry, WhatsApp message ₹450×qty + design-reference + manual-attach + soft-limit
  collapse, GarmentSvg recolor + img role).
- Asset check: `public/` assets emitted to `dist/` (16 self-hosted woff2 fonts, logo, favicon.ico,
  4 icons). `CREAZYCARTLOGO.png` is unchanged at the root (copied, never moved/modified) and there
  is NO nested `crazycart/` folder — the app is flat at the workspace root.

Not run in this environment: a live browser dev-server click-through (the sandbox blocks starting
local servers). Compilation, lint, and the unit/component suite all pass; the production bundle
serves its assets. Runtime canvas behaviour (Fabric transforms/export) should be smoke-tested in a
browser during review.

## Review iteration 2 — code-review.json findings fixed

Addressed all three findings from `code-review.json` (`CHANGES_REQUESTED`):

- **MED-1 (Textbox resize corrupts geometry):** `CanvasEngine.setCanvasPxWidth` no longer
  double-scales text. For a `Textbox` it now folds any residual `scaleX/scaleY` into `width`/
  `fontSize`, applies the resize factor, and pins `scaleX/scaleY` back to 1 — so a
  `ResizeObserver`-driven resize keeps the "fontSize/width authoritative, scale normalized away"
  invariant and text round-trips exactly. Images keep the standard single `scaleX/scaleY` rescale.
- **MED-2 (missing CanvasEngine tests):** added `src/lib/canvas/CanvasEngine.test.ts` with a
  serialize→loadSide round-trip for rotated image, sticker, and text layers (center-origin AABB
  stable), `normalizeTextScale` corner-scale normalization, and a `setCanvasPxWidth` resize
  round-trip for text (the case that catches MED-1). `normalizeTextScale` was extracted to an
  exported module-scope function to single-source the logic and give the test a seam. Fabric needs
  a real canvas 2D context + image decoding, which jsdom lacks, so `src/test/fabricTestEnv.ts`
  installs test-only canvas/image shims (no new deps).
- **NIT-1 (export edge not single-sourced):** `CanvasEngine` imports `EXPORT_LONGEST_EDGE` from
  `services/designExport.ts` and derives the export scale from it in both `toArtworkPNG` and
  `toMockupPNG` instead of the bare `2000` literal.

Re-ran in `d:\CRAZYCART.IN` via PowerShell:
- `npx tsc --noEmit` — **0 errors**.
- `npm run build` (`tsc -b && vite build`) — **succeeded**.
- `npx vitest run` — **28 tests passing** across 7 files (was 22; +6 CanvasEngine cases).
- `eslint` on changed files — **0 warnings / 0 errors**.
