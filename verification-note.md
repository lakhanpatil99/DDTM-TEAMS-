# CrazyCart.in — Rescue + Redesign Verification Note

This records the verification commands run in `d:\CRAZYCART.IN` (Windows / PowerShell)
and their results, so the reviewer can read the evidence without re-running the suites.

## Commands run and results

| Command | Expected | Result |
| --- | --- | --- |
| `npm install` | dependencies resolved | OK — up to date, 494 packages audited |
| `npx tsc --noEmit` | 0 errors | PASS — exit 0, no output |
| `npm run lint` (`eslint . --max-warnings 0`) | 0 warnings | PASS — exit 0, no warnings |
| `npm run build` (`tsc -b && vite build`) | success, Fabric code-split | PASS — built; `CanvasEngine` (Fabric) in its own chunk (~298 kB) off the marketing `index` chunk |
| `npx vitest run` | all pass incl. engine round-trip/normalization | PASS — 7 files, 30 tests passed |

### Build chunk evidence (Fabric stays off the marketing chunk)
```
dist/assets/index-*.css          34.14 kB
dist/assets/Review-*.js           9.20 kB
dist/assets/Studio-*.js          41.38 kB
dist/assets/CanvasEngine-*.js   298.60 kB   <- Fabric, lazy-loaded for the studio
dist/assets/index-*.js          447.30 kB
```

### Test coverage of the carry-over fixes (Part 1)
`npx vitest run src/lib/canvas` — `CanvasEngine.test.ts` (8 tests) + `geometry.test.ts` (4 tests):
- Serialize → `loadSide` round-trip of a rotated image/sticker layer (center-origin AABB stable) — MED-2.
- Textbox corner-scale normalization on commit (`normalizeTextScale`) — MED-2 / criterion 13.
- `setCanvasPxWidth` resize round-trip for text (no transient scale survives) — MED-1 / criterion 13.
- Live-canvas object presence with non-zero in-stage bounds + dispose-mid-load guard (editor render-pipeline fix).

`CanvasEngine.toArtworkPNG` / `toMockupPNG` derive the export scale from the shared
`EXPORT_LONGEST_EDGE` constant in `src/services/designExport.ts` — NIT-1.

## Hard invariants confirmed
- **No purple anywhere in the UI.** Tokens re-sourced to ivory/ink/forest/tangerine/lime/stone;
  legacy `purple`/`gold`/`cream`/`sand`/`line` class aliases migrated out and removed from
  `tokens.css` / `tailwind.config.ts`. The only remaining `purple` references are the legitimate
  selectable garment **fabric** colour (`src/data/colors.ts`) and a recolor unit-test assertion —
  no purple is rendered in chrome or home previews.
- **Frontend only.** No backend, database, auth, payment, or checkout.
- **WhatsApp handoff.** `wa.me/918208818421`, manual-attach copy, no false "order placed /
  message sent / image auto-attached" claims.
- **Pricing.** ₹450 single-sourced in `src/data/pricing.ts` (`baseDesignPrice: 450`); estimate =
  ₹450 × quantity; "confirm with business" note kept; no invented discounts/shipping/tax.
- **Logo.** `CREAZYCARTLOGO.png` unmodified at the workspace root; chrome places the logo on an
  ink/ivory surface (no purple logo background). No nested `crazycart/` folder.
- **Geometry contract preserved.** Fabric CENTER origin, layer `(x,y)` = box centre in template
  units, print-area `(x,y)` = top-left, rotated-AABB overflow, `sticker:<id>` never hitting
  IndexedDB, `EXPORT_LONGEST_EDGE = 2000`, Textbox normalize-on-commit.

## Part 6 — location
Business address and Google Maps link are single-sourced in `src/data/site.ts`
(`BUSINESS_ADDRESS`, `BUSINESS_MAPS_URL`) and surfaced on the Contact page ("Get directions",
opens in a new tab with `rel="noopener"`) and in the footer. No API-key embed — a plain outbound
maps link only; the Contact form still only drafts a WhatsApp/email message client-side.

_Verification run on the committed `master` branch after the token re-theme, chrome/logo fix,
editorial hero, contact location block, legacy-alias migration, and the editor render-pipeline +
studio/home/garment redesign landed via the task pipeline._
