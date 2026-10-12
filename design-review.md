# Design Review — CrazyCart.in (`design.md`)

Reviewed fresh, without the context that produced the document. Severity tags: **HIGH**, **MEDIUM**, **NIT** (NIT = LOW/informational-but-still-a-finding; it never affects the gate). The verdict is mechanical: any HIGH or MEDIUM ⇒ CHANGES_REQUESTED.

The document is unusually mature — it has already absorbed three prior review passes (§20, §20.1) and most of the brief's requirements map cleanly onto concrete, testable acceptance criteria (§17). Tokens, data model, pricing, WhatsApp handoff, FRONTEND-ONLY posture, and asset strategy are all concrete. The findings below are the gaps that remain, concentrated in the one area the whole studio rests on: the canvas geometry contract.

---

## Findings

### 1. HIGH — Rotation origin contradicts the "exact round-trip" geometry claim (§7.1 vs §9.4)

§7.1 defines layer geometry as: `(x,y)` is the **top-left of the axis-aligned, unrotated box**, and `rotationDeg` is "applied **about the box center**." But §9.4's load/commit mapping says, verbatim:

> **Load:** `left = x * s, top = y * s, angle = rotationDeg, origin top-left`
> **Commit:** `x = obj.left / s, y = obj.top / s, rotationDeg = obj.angle`

In Fabric.js an object with `originX:'left', originY:'top'` rotates **about its top-left corner**, not its center. So the two sections describe incompatible pivots. Concretely:

- With a top-left origin and nonzero `angle`, `obj.left/obj.top` is the rotated anchor, **not** the top-left of the unrotated box that §7.1 promises. Anything that reasons about geometry *outside* Fabric using the §7.1 semantic — the print-area out-of-bounds detection (§9.6), the artwork-only print-area crop (§8.2/§11.5), the bounds flag carried into the review summary — will misplace or mis-crop any **rotated** layer.
- The doc's headline guarantee ("this single-model rule … is what makes side-switch preservation and reopen **exact**") is therefore false for rotated layers, and criterion 13 explicitly requires rotations to be preserved exactly.

**Concrete fix:** pick one pivot and make both sections agree. Recommended: use Fabric **center origin** (`originX:'center', originY:'center'`) and define the stored `(x,y)` as the box center in template units; load with `left = (x)*s, top = (y)*s` and commit `x = obj.left/s, y = obj.top/s`. If `(x,y)` must stay the unrotated-box top-left, then load with center origin and convert: `left = (x + width/2)*s, top = (y + height/2)*s`, and on commit convert back from the center. Either way, state that the pivot used for `rotationDeg` and the pivot implied by `(x,y)` are the same point, and update §9.6/§11.5 bounds math to compute the rotated AABB from that single definition.

### 2. MEDIUM — Textbox corner-scaling is not folded into `fontSize`/`width`, so text size is lost on reload (§9.4)

§9.4 says the serialized schema "does **not** store Fabric's `scaleX/scaleY`" and, for text, commit reads back `fontSizeUnits = obj.fontSize / s` and `width = obj.getScaledWidth() / s`. This works for the image path (where `getScaledWidth` absorbs the scale and `fabricScale = width*s/naturalWidth` reproduces it on load). It does **not** work for a Fabric `Textbox` resized with its **corner** handles: corner handles change `scaleX/scaleY`, not `fontSize`. So `obj.fontSize / s` captures the *pre-scale* font size while the on-screen glyphs are `fontSize * scaleY` tall. On reload §9.4 sets `fontSize = fontSizeUnits * s` and discards the scale, so the text reloads at the wrong size — breaking the same "exact" guarantee for text (criterion 13 again).

**Concrete fix:** normalize text scale into font metrics on commit. On `scaling`/`modified` of a text object, set `fontSize = round(obj.fontSize * obj.scaleY)` and `width = obj.width * obj.scaleX`, then reset `obj.scaleX = obj.scaleY = 1` before reading geometry back — so the discarded `scaleX/scaleY` are genuinely absorbed, not dropped. Alternatively, disable corner (uniform-scale) controls for Textbox and allow only side handles (wrapping `width`) plus the `fontSize` field. State which, in §9.4.

### 3. MEDIUM — Sticker "recolor" is a named feature with no field in the data model to persist it (§8.5 vs §7.1)

§8.5 lists sticker capabilities including "recolor-where-the-SVG-allows," and §10.3/criterion 27d treat stickers as first-class editable layers. But stickers are serialized as `ImageLayer` (`type:'sticker'`), and `ImageLayer` in §7.1 has no `fill`/`tint`/color field. A sticker is referenced only by `blobKey` + `mimeType` + natural dimensions. So any recolor the user applies cannot be serialized and will not survive save→reopen or a side switch. Worse, if a sticker is rasterized to a bitmap blob (see finding 4), recolor isn't expressible at all.

**Concrete fix:** either (a) drop the recolor claim for stickers (keep move/resize/rotate/opacity/duplicate/delete/reorder — which the model already supports) and update §8.5; or (b) add an optional `fill?: string` (and the recolor rule — e.g. "sets `fill` on the sticker SVG's recolor-target element, applied at load before rasterizing") to a sticker-specific layer type, and specify that stickers are kept as inline/parsed SVG (not a flattened bitmap) so the fill can be reapplied. Pick one explicitly.

### 4. MEDIUM — Sticker storage uses an unresolved "X or Y" that has a correctness trap (§8.5)

§8.5: stickers are "bundled SVGs **promoted into IndexedDB on first use** (**or** referenced by a reserved `sticker:<id>` blobKey scheme the blob resolver understands)." This is two different persistence strategies left unchosen. They are not equivalent: the IDB-promotion branch means a **bundled** sticker's blob lives only in the user's IndexedDB, so clearing site data (or opening a draft shared/reloaded after an IDB wipe, or private-mode fallback per §11.3) turns a first-party sticker into the **missing-blob placeholder** of §11.3 — contradicting the "never broken images" posture and wasting the recovery UI on an asset that ships in the bundle. The reserved `sticker:<id>` scheme (resolver reads from the bundle, never from IDB) has no such failure mode.

**Concrete fix:** commit to the reserved `sticker:<id>` blobKey scheme: the blob resolver recognizes the `sticker:` prefix and loads the markup from `data/stickers.ts`/`src/assets/stickers/` directly, never from IndexedDB, so bundled stickers always resolve regardless of IDB state. Reserve IndexedDB promotion only for user-uploaded images. Remove the "or."

### 5. NIT — Export output resolution / `sₑ` is never pinned to a number (§11.5 vs §8.6)

§11.5 says exports render "at fixed export scale" and §9.4 references an export scale `sₑ` but gives no value; only §8.6 mentions "the 2000px export" in passing. For a build-ready spec the export dimension should be concrete and single-sourced.

**Concrete fix:** state it in §11.5, e.g. "export longest edge = 2000px ⇒ `sₑ = 2000 / viewBox.width` (width is the long axis for all §8.6 viewBoxes)," and make it a config constant so artwork-only and mockup exports share it.

### 6. NIT — Inlining the same garment SVG many times produces duplicate element ids in one document (§8.1/§8.6)

The category grid (8 cards), hero composition, and showcase all inline garment SVGs via `dangerouslySetInnerHTML`, each carrying `id="garment-body"` (and `id="garment-trim"`). Multiple identical ids in one document is invalid HTML; it's functionally safe here only because the component scopes its query to its own wrapper ref and the reference SVG uses no internal `url(#…)` references. The moment any overlay uses a gradient/filter/clipPath `url(#id)`, the duplicate ids will cross-wire.

**Concrete fix:** namespace or uniquify ids on inject (e.g. prefix every id with a per-instance token and rewrite internal `url(#…)` references), or state the invariant explicitly: "garment SVGs must never contain internal `url(#id)` references; recolor targets are matched by a `data-recolor="body|trim"` attribute scoped to the wrapper, not by document id."

### 7. NIT — Quantity direct-input validation behavior is unspecified (§12 / §4.5 Stepper)

§12 gives bounds (`minQuantity:1`, `maxQuantity:999`) and §4.5 lists a Stepper, but nothing specifies what happens when a user **types** an out-of-range or non-numeric value into the quantity field (clamp? reject? revert?), unlike the thorough rules given for upload (§10.1), text (§10.2), and rename (§11.2).

**Concrete fix:** add a one-liner to §12: "Quantity input is clamped to `[1,999]`; non-numeric/empty input reverts to the last valid value on blur; totals recompute from the clamped value."

---

## Verified Assumptions

- **Supplied logo is a raster PNG gold wordmark reading "CrazzyCart.in" with light/near-white lower text ("DTF & SUBLIMATION CUSTOM PRINTING HUB") on transparency.** Read the image directly. This confirms §4.1 / §8.4 / §16.1: it must sit on a dark/purple surface or a purple chip (light lower text is invisible on cream), and the "CrazzyCart" vs brand "CrazyCart" spelling mismatch is real and correctly handled.
- **Workspace is greenfield** — only `CREAZYCARTLOGO.png` and `.agents/` exist; no `package.json`, `src/`, or prior config (confirmed via directory listing + file search). So "preserve useful existing configuration/assets" reduces to preserving the one logo asset, which §3/§8.4 do (copy, never modify). Building flat at `d:\CRAZYCART.IN` per §3 is unobstructed.
- **WhatsApp target `wa.me/918208818421` correctly encodes +91 8208818421** (country code 91 + the 10-digit number). §13 is correct, uses dynamic selections, makes no auto-attach claim, explains manual attachment, and provides a download action — all verified against the brief's non-negotiables.
- **FRONTEND-ONLY is honored** — §2 and §18 exclude backend/DB/auth/payment/server order processing; no fake order/success screen; WhatsApp is the only handoff. Consistent throughout.
- **₹450 is single-sourced** in `data/pricing.ts` with `basePriceOverride?: never` forbidding per-product surcharges (§6.1), shown consistently, with an honest disclaimer and no invented discounts/tax/shipping (§12). Verified.
- **Vite `?raw` import, the `idb` package, and the named OFL fonts (Inter, Fraunces, Montserrat, Poppins, Oswald, Bebas Neue, Pacifico, Lobster) are all real and acquirable as self-hosted woff2.** No fictional dependency.

## Unverified / Wrong Assumptions

- **WRONG (as written): Fabric top-left origin reproduces a center-pivot rotation** (finding 1). Fabric rotates about the configured origin; top-left origin + `angle` pivots about the corner, contradicting §7.1's center-rotation semantic.
- **UNVERIFIED → likely a gap: Fabric v6 Textbox corner-scaling folds into `fontSize`** (finding 2). Fabric Textbox corner handles mutate `scaleX/scaleY`; the design's commit path doesn't absorb them, so stored text size will disagree with the rendered size unless normalization is added. The implementer must verify Textbox control behavior in the installed `fabric` v6 and wire the normalization.
- **UNVERIFIED (handled): `mix-blend-mode` is unreliable when an SVG is rasterized to canvas on iOS WebKit** (§11.5/§9.7). The design *assumes* this and ships a `globalCompositeOperation` fallback, so even if the assumption is partly wrong it degrades gracefully — not a finding, but flagged because it's an assumption the implementer must test on-device.
- **UNVERIFIED: the exact Fabric v6 API surface** used in §9.1/§9.4 (`getScaledWidth/getScaledHeight`, `Textbox`, serialization) — nothing is installed yet, so this is taken from general knowledge. The `CanvasEngine` abstraction (§9.1) correctly contains the risk to one file, which is the right mitigation.

---

## Verdict

Findings: **1 HIGH, 3 MEDIUM, 3 NIT.** HIGH+MEDIUM count = 4 (> 0) ⇒ **CHANGES_REQUESTED.**

The remaining work is narrow and all solvable with text edits — it is concentrated in the canvas geometry contract (findings 1–2), which is load-bearing for the studio's "exact preservation" guarantees, plus two sticker-model gaps (3–4). Resolve those four and the document is ready to build against.
