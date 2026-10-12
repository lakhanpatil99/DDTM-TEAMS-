# CrazyCart.in — Requirements & Technical Design

> Single source of truth for the planner and implementers. The stack is locked (see §2). Everything below is concrete enough to implement without further architectural decisions. Where the brief was ambiguous, §16 records the resolution that was adopted and already baked into this document.

---

## 1. Summary

CrazyCart.in is a **frontend-only**, production-quality, premium custom-printing e-commerce experience for a DTF-printing / sublimation studio. It lets a visitor browse a data-driven catalog, open an in-browser **custom design studio**, configure a garment (product, color, size, material, print locations), add and manipulate artwork and text on a Fabric.js canvas, preview the result, persist drafts locally (IndexedDB + localStorage), export a preview image, and hand off the finished design to the business over **WhatsApp click-to-chat**.

There is **no backend**: no APIs, no database server, no payment gateway, no auth, no server-side order processing, and no fake order/payment success screens. WhatsApp is the one and only handoff. Image attachment is manual — the app downloads the preview and tells the user to attach it in WhatsApp; it never claims an image auto-attaches or that an order was received.

The visual language is editorial and premium: **deep purple** accent, **warm golden-yellow** highlights (matching the gold wordmark logo), off-white / soft-neutral backgrounds, deep-charcoal type, restrained gradients, generous whitespace, rounded cards with subtle borders and controlled shadows. Motion is intentional and respects `prefers-reduced-motion`.

**Brand facts (locked):** Name CrazyCart.in · Tagline "Shaping Ideas with Design" · Business DTF printing, sublimation, custom T-shirts/caps/personalized products · Base design price **₹450** per customized product (central config) · WhatsApp **+91 8208818421** → wa.me target **918208818421**.

---

## 2. Technology Stack (LOCKED)

| Concern | Choice | Notes |
|---|---|---|
| Framework | **React 18 + TypeScript** | Function components + hooks only |
| Build/dev | **Vite** | `strict` TS, path alias `@/` → `src/` |
| Styling | **Tailwind CSS** | Centralized design tokens via CSS variables + `tailwind.config` theme extension |
| Animation | **Framer Motion** | Must respect `prefers-reduced-motion`; animate transform/opacity only |
| Canvas editor | **Fabric.js** (`fabric` v6) | **Committed.** Konva is NOT used. See §9 for the rationale and the abstraction that isolates us if Fabric ever needs replacing |
| Routing | **React Router** (v6 data router / `createBrowserRouter`) | Client-side only |
| Local blob storage | **IndexedDB** via **`idb`** package | Artwork image blobs + exported preview blobs |
| Local metadata storage | **localStorage** | Small design metadata, draft index, UI prefs. **Never** store large Base64 artwork here |
| State | React Context + `useReducer` per domain (studio, cart/selection, saved-designs) + local component state | No Redux. See §9.3 |
| Icons | **lucide-react** | Tree-shakeable, consistent stroke icons |
| Package manager | **npm** | Lockfile committed |
| Testing | **Vitest** + **@testing-library/react** (unit/component); **Playwright** (optional e2e smoke) | See §15 |
| Lint/format | ESLint (typescript-eslint, react-hooks, jsx-a11y) + Prettier | Zero-warning policy on CI |

Node ≥ 18. No SSR, no Next.js. The app is a static SPA deployable to any static host.

---

## 3. Project Structure

**The app is built FLAT at the workspace root** `d:\CRAZYCART.IN` — `package.json`, `index.html`, `src/`, `public/` all live directly here. **Do NOT create a nested `crazycart/` (or `crazycart/crazycart/`) subfolder.** The structure below adopts the suggested architecture from the supplementary requirements (`pages/`, `data/`, `hooks/`, `services/`, `types/`, `utils/`, `styles/`, `components/{layout,home,products,editor,review,ui}`), adapted so no empty stub files are created — every listed module has real responsibilities defined in this doc.

```
d:\CRAZYCART.IN\
├─ CREAZYCARTLOGO.png              # ORIGINAL supplied asset — NEVER modify or delete
├─ index.html
├─ package.json  package-lock.json
├─ README.md                      # deliverable (§19): install/dev/build + asset limitations
├─ tsconfig.json  tsconfig.node.json
├─ vite.config.ts
├─ tailwind.config.ts  postcss.config.js
├─ .eslintrc.cjs  .prettierrc  .gitignore
├─ public/
│  ├─ assets/logo/                # COPY of CREAZYCARTLOGO.png (+ processed variants); used by header/footer + favicon source
│  ├─ favicon.ico / icons (16/32/180/512 generated from the logo)
│  └─ fonts/                      # self-hosted woff2 (see §4.2)
└─ src/
   ├─ main.tsx  App.tsx  router.tsx
   ├─ styles/
   │  ├─ tokens.css               # CSS variables (design tokens, §4)
   │  ├─ globals.css              # base, resets, @font-face, utilities
   │  ├─ animations.css           # keyframes / reduced-motion fallbacks
   │  └─ responsive.css           # shared responsive helpers (safe-area, dvh)
   ├─ assets/
   │  ├─ garments/                # SVG garment templates per product/side (§8)
   │  └─ stickers/                # curated local SVG sticker set (§8.5)
   ├─ data/                       # data-driven config (plain .ts modules)
   │  ├─ products.ts  categories.ts  colors.ts  fonts.ts  stickers.ts
   │  ├─ pricing.ts               # central pricing (§12)
   │  └─ site.ts                  # brand strings, WhatsApp number, nav, social, FAQ
   ├─ types/
   │  ├─ product.ts  design.ts  editor.ts
   ├─ services/
   │  ├─ designStorage.ts         # idb + localStorage repo, auto-save, rename, quota/corruption (§11)
   │  ├─ designExport.ts          # PNG export (artwork-only + mockup) (§11.3)
   │  └─ whatsapp.ts              # message builder + wa.me URL (§13)
   ├─ lib/canvas/                 # Fabric abstraction: CanvasEngine (§9)
   ├─ utils/
   │  ├─ pricing.ts  validation.ts  imageProcessing.ts  formatting.ts
   │  └─ cn.ts  id.ts
   ├─ hooks/
   │  ├─ useDesign.ts  useLocalDrafts.ts  useUndoRedo.ts  useMediaQuery.ts
   │  └─ usePrefersReducedMotion.ts  useScrollDirection.ts  useDocumentTitle.ts
   ├─ context/ (StudioProvider, SelectionProvider, SavedDesignsProvider, ToastProvider)
   ├─ components/
   │  ├─ ui/                      # Button, Card, Input, Modal, Drawer, BottomSheet, Swatch, Badge, Tabs, Tooltip, Toast, Accordion, Stepper, SizeGuideModal, ConfirmDialog, Skeleton
   │  ├─ layout/                  # AnnouncementBar, Header, Nav, MobileDrawer, Footer, Container, SectionReveal
   │  ├─ home/                    # Hero, ProductShowcase, CategoryGrid, FeaturedProducts, StudioShowcase, HowItWorksTeaser, PrintingInfo, MerchShowcase, FaqAccordion, ContactCta
   │  ├─ products/                # ProductCard, ColorSwatches, SizeSelector, MaterialSelector, PrintLocationTabs, FilterBar, MobileFilterDrawer, QuickPreviewModal, PriceSummary
   │  ├─ editor/                  # StudioLayout, ProductConfigPanel, CanvasStage, PropertiesPanel, LayersPanel, DesignToolbar, MobileToolbarSheet, StickerPicker, DesignSummary
   │  └─ review/                  # ReviewScreen, WhatsAppHandoff, DownloadPreview
   └─ pages/
      Home · Shop · ProductDetail · Studio · Review · SavedDesigns · HowItWorks · About · Contact · NotFound
      # NOTE: there is intentionally NO separate Catalog page component. The
      # routes /shop and /catalog both render the SAME Shop page (§5); /catalog
      # is the deep-link/filtered alias used by category cards and the hero
      # "Explore Products" CTA. One component, two paths — see §5.
```

---

## 4. Design System

All tokens live in `src/styles/tokens.css` as CSS custom properties and are mirrored into `tailwind.config.ts` so Tailwind utilities (`bg-brand-purple`, `text-ink`, `rounded-card`, `shadow-card`) resolve to the same values. **No hard-coded hex outside the token files.**

### 4.1 Color tokens

Brand accent is deep purple; highlight is the logo's golden-yellow. Backgrounds are warm off-white neutrals; text is deep charcoal. Purple is an **accent**, not a background wash (per brief: avoid excessive purple backgrounds).

```
/* Brand */
--purple-50:#f4f1fb  --purple-100:#e6def6  --purple-200:#cdbcee  --purple-300:#ad92e2
--purple-400:#8c67d4  --purple-500:#6d3fc4  --purple-600:#5a2ea8  /* primary accent */
--purple-700:#48228a  --purple-800:#361a66  --purple-900:#241144

--gold-50:#fdf8ec   --gold-100:#f9edc9  --gold-200:#f2db94  --gold-300:#ecc85c
--gold-400:#e4b73a  /* highlight */     --gold-500:#cf9f23  --gold-600:#a97e18

/* Neutrals (warm) */
--neutral-0:#ffffff --cream:#faf7f2 /* page bg */ --sand:#f3eee6 /* alt section bg */
--neutral-200:#e7e2d9 --neutral-300:#d6cfc2 --neutral-400:#a9a195
--ink:#1f1b24 /* primary text, charcoal w/ purple undertone */
--ink-soft:#4a4550  --ink-muted:#726c78  --line:#e9e4db /* hairline borders */

/* Semantic */
--success:#2f8f5b  --warning:#c7881f  --danger:#c23b3b  --info:#3b6fc2
--focus-ring:#8c67d4  /* purple-400; rendered as the 3px --shadow-focus ring (§4.3) */

/* Gradients (restrained, used sparingly) */
--grad-brand: linear-gradient(135deg, var(--purple-700), var(--purple-500));
--grad-gold:  linear-gradient(135deg, var(--gold-400), var(--gold-200));
--grad-hero-wash: radial-gradient(1200px 600px at 80% -10%, rgba(109,63,196,.10), transparent 60%);
```

**Dark surface for the logo:** the supplied logo is a **raster gold wordmark** with light lower text, so it is never placed directly on cream. **Robust default (shipped):** render the original gold logo inside a small `--purple-800` rounded chip (`rounded-sm`, subtle padding + `--shadow-sm`) in the light header, and the original gold logo directly on `--purple-800` in the footer and any purple section. This keeps the asset legible everywhere with zero image editing. A truly recolored purple/ink **vector** wordmark is treated as an **optional later asset swap** (requires manual redraw/masking, not a CSS step), not a build dependency — the header reads a single `logoVariant` config and already works with the chip default. Processed assets live in `src/assets/logo/` (see §8.4).

Contrast: body text (`--ink` on `--cream`) and all interactive text must meet WCAG AA (≥4.5:1 normal, ≥3:1 large). Gold is a **highlight/decoration**, never used for small body text on light backgrounds (fails contrast); for gold emphasis use `--gold-600` on light or gold on purple.

### 4.2 Typography

Editorial display face + readable sans, self-hosted woff2 in `public/fonts/` with system fallbacks (no FOIT; `font-display:swap`). Load only the weights listed.

- **Display:** `Fraunces` (variable, opsz/wght) — editorial, high-contrast serif for headings/hero. Fallback: `"Fraunces", "Playfair Display", Georgia, serif`.
- **Sans (body/UI):** `Inter` (variable) — Fallback: `"Inter", system-ui, -apple-system, "Segoe UI", Roboto, Arial, sans-serif`.

**Two font sets, both self-hosted.** The site chrome (marketing + studio UI) uses only **Fraunces + Inter** (the weights/axes above). The studio **text tool** offers a larger curated set that the user can apply to their artwork (§10.2); those display faces are self-hosted the same way (woff2 in `public/fonts/`, `font-display:swap`) but are **used only inside the canvas**, never for site UI, so they add no render-blocking cost to marketing pages (they are loaded lazily on `/studio`). Font loading for the canvas is covered in §10.2 (Fabric must not render/export text before the chosen face is actually loaded).

**Type scale.** Each step is a **separate size token plus its own line-height / letter-spacing tokens** (the `size/lh` notation below is doc shorthand only — never emit `clamp(...)/1.05` as a single CSS custom-property value, it is invalid outside the `font:` shorthand). In `tokens.css` each row becomes e.g. `--fs-display:clamp(2.75rem,6vw,4.5rem); --lh-display:1.05; --ls-display:-0.02em;`. In `tailwind.config.ts` the same values are expressed as a Tailwind `fontSize` tuple: `display: ['clamp(2.75rem,6vw,4.5rem)', { lineHeight: '1.05', letterSpacing: '-0.02em' }]`.

```
--font-display: fraunces stack       --font-sans: inter stack
/* Type scale (fluid via clamp, 1.250 major-third desktop) — shown as size / line-height */
--fs-display: clamp(2.75rem, 6vw, 4.5rem)  --lh-display:1.05  --ls-display:-0.02em  (weight 600)
--fs-h1: clamp(2.1rem,4vw,3rem)  --lh-h1:1.1      --fs-h2: clamp(1.6rem,3vw,2.25rem)  --lh-h2:1.15
--fs-h3: 1.5rem  --lh-h3:1.2      --fs-h4:1.25rem  --lh-h4:1.25
--fs-body-lg:1.125rem --lh-body-lg:1.6   --fs-body:1rem --lh-body:1.6
--fs-sm:0.875rem --lh-sm:1.5           --fs-xs:0.75rem --lh-xs:1.45
--fw-regular:400 --fw-medium:500 --fw-semibold:600 --fw-bold:700
```
Headings use `--font-display`; body, UI, labels, buttons use `--font-sans`. The studio UI uses sans exclusively (dense, functional).

### 4.3 Spacing, radii, shadows, layout

```
/* Spacing scale (4px base) */ --space-1:4 --2:8 --3:12 --4:16 --5:20 --6:24 --8:32 --10:40 --12:48 --16:64 --20:80 --24:96 (px)
/* Radii */ --radius-xs:6 --sm:10 --md:14 --lg:20 (cards) --xl:28 --pill:999 (px)  -> token names rounded-card = lg
/* Shadows (controlled, soft, purple-tinted) */
--shadow-xs: 0 1px 2px rgba(31,27,36,.06)
--shadow-sm: 0 2px 8px rgba(31,27,36,.06)
--shadow-card: 0 8px 24px rgba(31,27,36,.08)
--shadow-lg: 0 18px 48px rgba(36,17,68,.14)
--shadow-focus: 0 0 0 3px rgba(140,103,212,.45)
/* Layout */ --container-max:1240px --container-pad: clamp(16px,4vw,40px)
--header-h:72 (desktop) /64 (mobile)   --z: header 50, drawer 60, bottomsheet 70, modal 80, toast 90
```

### 4.4 Motion tokens

```
--ease-standard: cubic-bezier(.2,.8,.2,1)   --ease-emphasized: cubic-bezier(.2,.9,.1,1)
--dur-fast:150ms --dur-base:250ms --dur-slow:400ms --dur-page:350ms
--stagger:60ms (card reveals)
```
All motion is gated by a `usePrefersReducedMotion()` hook: when reduced, durations collapse to 0 / cross-fades only, no transforms, no parallax, no scroll-reveal translation (content appears immediately). See §14.

### 4.5 Component primitives (`components/ui`)

Button (variants: `primary` purple, `gold` highlight CTA, `secondary` outline, `ghost`, `danger`; sizes sm/md/lg; loading + disabled states; min touch target 44×44px), Card, Badge/Label (e.g. "Bestseller" purely descriptive, no fake stock), Swatch (color, with selected ring + checkmark + `aria-pressed`), Input/Textarea/Select/Range (labeled, error text), Modal (focus-trapped, Esc, overlay click, `role="dialog" aria-modal`), Drawer (mobile nav), BottomSheet (mobile studio tools, drag handle, snap points), Tabs, Tooltip, Toast (save/validation feedback), SegmentedControl, Stepper (quantity), SizeGuideModal, SkeletonLoader. Every interactive primitive is keyboard-operable with a visible `--shadow-focus` ring.

---

## 5. Pages & Route Map

Client-side routes (`createBrowserRouter`). A root layout renders `Header` + `Footer` + animated `<Outlet/>`; the Studio uses a **dedicated full-bleed layout without the marketing footer**.

| Route | Page | Purpose / key content |
|---|---|---|
| `/` | **Home** | Full editorial landing — section list in §5.2 |
| `/shop` `?category=&color=&material=&style=&sort=&q=` | **Shop** | Search + filters + sort + quick preview over the full catalog, all filter state URL-synced — see §5.3 |
| `/catalog` `?category=&color=&material=&style=&sort=&q=` | **Shop (alias)** | **Renders the exact same `Shop` page component** — NOT a distinct page. `/catalog` is the deep-link/filtered alias that category cards (§5.4) and the hero "Explore Products" CTA (§5.1) target; both paths mount `<Shop/>` with identical query-param handling. There is no separate `Catalog` module to build |
| `/product/:productId` | **Product detail** | Large preview, color/size/material/print-location selectors, price summary (₹450 base), product info, care instructions (if supplied), "Customize This Product" CTA |
| `/studio` `?product=&color=&size=&material=&draft=` | **Design Studio** | The core feature (§9, §10). Deep-linkable with a preselected product/config or an existing draft id |
| `/studio/review` | **Review & Submit** | Final review screen → download + WhatsApp handoff (§13). Guards: redirects to `/studio` if no active design |
| `/saved` | **Saved Designs** | Draft cards with mockup thumbnail, config summary, last-saved time; reopen, **rename**, delete-with-confirm, empty state, missing-blob recovery messaging — see §11.5 |
| `/how-it-works` | **How It Works** | Steps: Choose → Design → Preview → Send on WhatsApp → Confirm with team |
| `/about` | **About** | Brand story, DTF/sublimation explainer, "Shaping Ideas with Design" |
| `/contact` | **Contact** | WhatsApp CTA (wa.me), `mailto:` link, business hours, socials. **No backend form.** If a form is shown, it composes a `mailto:`/WhatsApp message client-side only |
| `*` | **NotFound** | Branded 404 with route-home CTA |

Scroll restoration on navigation; `<title>`/meta per route via a small `useDocumentTitle` hook. Nav items: Home, Shop, Customize (→ `/studio`), How It Works, About, Contact + prominent **"Start Designing"** (→ `/studio`) button. Header is sticky + scroll-aware (condenses / hides-on-scroll-down, shows-on-scroll-up) with an optional compact **selection indicator** (shows active studio product/draft if one exists). Mobile uses a slide-in Drawer.

### 5.1 Hero (exact copy, locked)
- Headline: **"Your Ideas. Your Style. Your Creation."** (display face). The gold-accented key word on the light cream hero uses **`--gold-600` (#a97e18, ~3.6:1 on cream — passes AA large text)**, or sits on a small `--purple-700`/dark chip; it **never** uses `--gold-300/400` for text on light (those fail contrast per §4.1).
- Sub-copy: "Turn your imagination into wearable art. Customize T-shirts, caps, and more with your own designs, colors, and personal touch."
- Primary CTA **"Create Your Design"** → `/studio`. Secondary CTA **"Explore Products"** → `/catalog`.
- Visual: composition of SVG garment mockups + floating design-preview cards with tasteful entrance motion and subtle (reduced-motion-safe) parallax. **Not** an oversized empty hero — content-dense, balanced on mobile. No broken image URLs; uses the SVG garment system (§8).

### 5.2 Home page section order (locked)

**Layout-level chrome (rendered by the root layout on EVERY route, not Home modules — listed here only for top-to-bottom context so they are not double-rendered on `/`):**
- **L1. Announcement bar** — thin dismissible strip (e.g. "Custom DTF & sublimation printing — designs from ₹450. Chat with us on WhatsApp."), a `components/layout/AnnouncementBar`. Dismiss state is stored in `cc.prefs` **keyed by announcement id** (`prefs.dismissedAnnouncementId`), so publishing a new announcement id re-shows the bar even if a prior one was dismissed (storage key enumerated in §11.1).
- **L2. Premium nav / header** (§5 header rules), `components/layout/Header`.

**Home page modules** (`components/home`, each rendered top-to-bottom only on `/`; scroll-reveal with reduced-motion fallback):
3. **Editorial hero** (§5.1).
4. **Animated product showcase** — motion-driven carousel/marquee of garment mockups in varied colors (transform/opacity only; pauses on reduced-motion).
5. **Shop by category** — the 8 category cards (§5.4).
6. **Featured products** — subset of catalog (`heroImageOrder`), each card "Designs from ₹450" + "Create your own design".
7. **Design-studio showcase** — visual explainer of the studio with a "Start Designing" CTA → `/studio`.
8. **DTF & sublimation printing info** — editorial section explaining the two techniques and when each is used (informational, no fulfillment promises).
9. **Personalized merchandise showcase** — caps/totes/custom-merch highlight.
10. **FAQ accordion** — accessible disclosure (`components/ui/Accordion`), content from `data/site.ts` (honest answers: pricing basis, how WhatsApp handoff works, that mockups are previews, manual image attachment).
11. **WhatsApp / contact CTA band** — prominent wa.me CTA + mailto.
12. **Premium footer** — gold logo on `--purple-800`, nav, socials, tagline, copyright.

### 5.3 Shop page specifics
Search box (`q`, debounced, matches name/tags/category), category/color/style filter controls, sort (featured, name A–Z/Z–A, newest), **quick-preview modal** (opens a product mockup + key config + "Customize" without leaving the page), **mobile filter drawer** (bottom-sheet/side drawer consolidating filters on small screens), branded empty state when nothing matches, and consistent ₹450 pricing on every card. All filter/search/sort state is URL-synced for shareable/back-safe views.

### 5.4 Shop-by-category cards
8 cards: Custom T-shirts, Oversized T-shirts, Polo T-shirts, Sweatshirts, Hoodies, Caps, Tote bags, Custom merchandise. Each: SVG garment mockup (category's `previewProductId`+`previewColorId`), name, concise description, hover interaction; click → `/catalog?category=<id>` (or `/studio?product=` to preselect in the studio). Copy must not imply guaranteed physical fulfillment of every category.

---

## 6. Product, Color & Pricing Data Model (data-driven config)

All catalog content is config-driven in `src/data/*` and typed in `src/types/`. Adding/editing a product = editing config only. **No prices, colors, or product strings are hard-coded in components.**

### 6.1 Core TypeScript types (`src/types/product.ts`)

```ts
export type ProductCategoryId =
  | 'tshirt' | 'oversized-tshirt' | 'polo-tshirt' | 'sweatshirt'
  | 'hoodie' | 'cap' | 'tote-bag' | 'custom-merch';

export type PrintSide = 'front' | 'back' | 'left-sleeve' | 'right-sleeve';

// Compile-time safety for color references (mirrors the §6.2 palette). The
// dev-time validator (§6.3) still runs, but bad ids now also fail typecheck.
export type GarmentColorId =
  | 'black' | 'white' | 'navy' | 'red' | 'grey'
  | 'green' | 'yellow' | 'purple' | 'pink' | 'beige';

export type MaterialId =
  | 'cotton-180' | 'cotton-combed-220' | 'poly-dryfit'
  | 'cotton-poly-blend' | 'fleece-320' | 'canvas';

export interface Material {
  id: MaterialId;
  name: string;            // "100% Combed Cotton (220 GSM)"
  description?: string;
  recommendedPrint?: ('DTF' | 'Sublimation')[]; // informational only
}

export interface SizeOption {
  id: string;              // 'S' | 'M' | ... | 'OS' (one-size)
  label: string;           // display
  // measurements OPTIONAL; when absent UI marks size guide as placeholder
  chestInches?: number;
  lengthInches?: number;
}

export interface GarmentColor {
  id: GarmentColorId;      // 'black' | 'white' | 'navy' | ...
  name: string;            // display name
  hex: string;             // EXACT configured value, used to fill the SVG garment
  // optional dedicated raster image per side if a real photo asset exists later
  images?: Partial<Record<PrintSide, string>>;
  // text/UI contrast hint so overlaid labels stay legible on the swatch
  isLight: boolean;
}

/** A rectangular (optionally rounded) printable region, expressed in the
 *  garment template's own coordinate space (the SVG viewBox), so it scales
 *  with the preview. */
export interface PrintArea {
  side: PrintSide;
  label: string;           // "Front — A4 area"
  // bounds in template viewBox units
  x: number; y: number; width: number; height: number;
  rotationDeg?: number;    // for angled areas (rare)
  // real-world note shown to user (configurable, not a print guarantee)
  maxPrintNote?: string;   // e.g. "approx A4 (210×297mm)"
}

export interface GarmentTemplate {
  // SVG template per side; the color fill target is matched by a scoped
  // data-recolor attribute, NOT a document id (§8.1 duplicate-id invariant)
  side: PrintSide;
  svg: string;             // raw SVG markup (Vite `?raw` import), inlined via
                           // dangerouslySetInnerHTML so [data-recolor] is query-able (§8.1)
  viewBox: { width: number; height: number }; // per-side, from the §8.6 aspect table — NOT a global 1000x1200
  // the data-recolor attribute VALUE whose element gets the body color
  // (e.g. 'body'); matched scoped to the instance wrapper, never a global id
  colorTargetId: string;
}

export interface Product {
  id: string;              // 'classic-tshirt'
  categoryId: ProductCategoryId;
  name: string;
  shortDescription: string;
  longDescription?: string;
  careInstructions?: string[];   // optional; shown only if present
  styleTags: string[];           // for the 'style' filter: 'unisex','relaxed','streetwear'...
  colors: GarmentColorId[];      // colors available for THIS product (compile-time checked)
  sizes: SizeOption[];           // only these sizes shown for this product
  materials: MaterialId[];       // empty => material selector hidden
  supportedSides: PrintSide[];   // only these print locations shown
  templates: GarmentTemplate[];  // one per supportedSide
  printAreas: PrintArea[];       // one per supportedSide
  // pricing: base is central; product may NOT invent its own surcharges
  basePriceOverride?: never;     // intentionally forbidden by type (keeps ₹450 single-source)
  badges?: string[];             // descriptive labels only (no fake stock/sale)
  heroImageOrder?: number;       // featured ordering
}

export interface ProductCategory {
  id: ProductCategoryId;
  name: string;                  // "Oversized T-shirts"
  description: string;
  previewColorId: GarmentColorId; // which color to render the category card in
  previewProductId: string;      // product used for the card mockup
}
```

`data/products.ts` exports `products: Product[]`, `data/categories.ts` exports `categories: ProductCategory[]`; `data/colors.ts` exports the ten+ named `GarmentColor`s (Black, White, Navy blue, Red, Grey, Green, Yellow, Purple, Pink, Beige) with **exact hex** values. The config module runs a dev-time validation (§6.3).

### 6.2 Colors (exact configured palette, `data/colors.ts`)

```
black  #1a1a1a (isLight:false)   white  #f7f7f5 (true)   navy #1f2d4d (false)
red    #c0392b (false)           grey   #8c8c8c (true)   green #2e7d4f (false)
yellow #e4b73a (true)            purple #5a2ea8 (false)  pink  #e59bb8 (true)
beige  #d9c9a8 (true)
```
(These are garment colors — distinct from the UI brand tokens, though purple/yellow intentionally echo the brand.) Selecting a swatch sets `fill` on the garment SVG `colorTargetId` instantly; the selected swatch shows a ring + check and `aria-pressed=true`.

### 6.3 Config validation (dev-time guard)
On module load in dev, assert: every `product.colors` id exists in the color palette; every `supportedSide` has exactly one matching `template` and one matching `printArea`; every `sizes` entry is unique; `materials` ids resolve; every `category.previewProductId` resolves to a product in that category and `previewColorId` is one of that product's colors. Violations `throw` in dev (fail fast) and are stripped in prod builds. This keeps the data-driven catalog self-consistent.

### 6.4 Concrete starter catalog (the ship-with content)

The build ships with **exactly these 8 products, one per category** (the garment-SVG inventory in §8 is scoped directly from this list — see §8.6). Shared size sets are defined once and reused:

```ts
// shared size presets (SizeOption[])
const APPAREL = ['S','M','L','XL','2XL','3XL'];        // tees, polo, sweatshirt, hoodie, oversized
const ONE_SIZE = ['OS'];                                // caps, totes, generic merch
// shared color presets (GarmentColor ids from §6.2)
const FULL10 = ['black','white','navy','red','grey','green','yellow','purple','pink','beige'];
```

| `id` | `categoryId` | `name` | `colors` | `sizes` | `materials` | `supportedSides` | `heroImageOrder` | `styleTags` |
|---|---|---|---|---|---|---|---|---|
| `classic-tshirt` | `tshirt` | Classic Unisex Tee | FULL10 | APPAREL | `cotton-180`, `cotton-combed-220` | `front`, `back`, `left-sleeve`, `right-sleeve` | 1 | unisex, relaxed, everyday |
| `oversized-tee` | `oversized-tshirt` | Oversized Drop-Shoulder Tee | black, white, navy, grey, green, beige, purple | APPAREL | `cotton-combed-220` | `front`, `back` | 2 | oversized, streetwear, unisex |
| `polo-tee` | `polo-tshirt` | Pique Polo Shirt | black, white, navy, red, grey | S–XL,2XL (no 3XL) | `cotton-poly-blend`, `poly-dryfit` | `front`, `back`, `left-sleeve` | 3 | smart-casual, corporate |
| `crew-sweatshirt` | `sweatshirt` | Crewneck Sweatshirt | black, white, navy, grey, beige | APPAREL | `fleece-320`, `cotton-poly-blend` | `front`, `back` | 5 | cozy, unisex |
| `pullover-hoodie` | `hoodie` | Pullover Hoodie | black, navy, grey, green, purple, beige | APPAREL | `fleece-320` | `front`, `back` | 4 | streetwear, cozy, unisex |
| `classic-cap` | `cap` | Structured Baseball Cap | black, white, navy, red, grey, beige | ONE_SIZE | *(none — selector hidden)* | `front` | 6 | adjustable, headwear |
| `canvas-tote` | `tote-bag` | Canvas Tote Bag | white, black, navy, beige | ONE_SIZE | `canvas` | `front`, `back` | 7 | eco, everyday |
| `custom-merch-mug` | `custom-merch` | Custom Printed Mug | white, black | ONE_SIZE | *(none — selector hidden)* | `front` | 8 | giftable, sublimation |

Notes baked into the config:
- **Polo** uses size set `['S','M','L','XL','2XL']` (no 3XL) and demonstrates per-product size restriction (criterion 11).
- **Cap** and **Mug** define empty `materials` ⇒ the material selector is **hidden** for them (§16.6).
- Every product's `printAreas` has exactly one `PrintArea` per `supportedSide` (§6.3 enforces this). Starter print-area notes: tee/sweatshirt/hoodie front/back ≈ "A4 area (210×297mm)"; sleeve ≈ "sleeve strip (90×60mm)"; cap front ≈ "cap front (120×55mm)"; tote ≈ "centered (250×300mm)"; mug ≈ "wrap panel (200×85mm)" — all marked configurable, not print guarantees.
- `badges` are descriptive only where set (e.g. `classic-tshirt`: `['Bestseller']`, `oversized-tee`: `['New']`) — no stock/sale semantics (§6.1, criterion 9).

**Category config (`data/categories.ts`), one row per card (§5.4), each pointing at its starter product:**

| `id` | `name` | `previewProductId` | `previewColorId` |
|---|---|---|---|
| `tshirt` | Custom T-shirts | `classic-tshirt` | `purple` |
| `oversized-tshirt` | Oversized T-shirts | `oversized-tee` | `beige` |
| `polo-tshirt` | Polo T-shirts | `polo-tee` | `navy` |
| `sweatshirt` | Sweatshirts | `crew-sweatshirt` | `grey` |
| `hoodie` | Hoodies | `pullover-hoodie` | `black` |
| `cap` | Caps | `classic-cap` | `red` |
| `tote-bag` | Tote bags | `canvas-tote` | `beige` |
| `custom-merch` | Custom merchandise | `custom-merch-mug` | `white` |

Each category's one-line `description` lives in `data/categories.ts` (honest, no fulfillment promise, e.g. tote: "Everyday canvas totes, printed with your art."). **Featured products** (§5.2.6) render the subset sorted by `heroImageOrder`. Adding more products later = appending rows to `data/products.ts` (and their garment SVGs); no component changes.

---

## 7. Design (Layer) & Saved-Design Data Model

### 7.1 Layer types (`src/types/design.ts`)

```ts
export type LayerType = 'image' | 'text' | 'sticker';

export interface BaseLayer {
  id: string;
  type: LayerType;
  side: PrintSide;         // which garment side this layer belongs to
  // AUTHORITATIVE GEOMETRY. All values are in the garment template's own
  // viewBox units (NOT screen px), so geometry is resolution-independent and
  // survives zoom/export/reopen. (x,y) is the CENTER of the layer's
  // axis-aligned (unrotated) box; (width,height) is the rendered box in
  // template units; rotation is applied about that same box center.
  // Storing the CENTER (not the top-left) matches Fabric's CENTER origin used
  // at load/commit (§9.4) so the pivot is identical in both representations and
  // rotated layers round-trip exactly (HIGH-1). Any top-left needed for bounds
  // is derived as (x - width/2, y - height/2); overflow/crop use the rotated
  // AABB computed from (x,y,width,height,rotationDeg) — single definition (§9.6/§11.5).
  // There is deliberately NO scaleX/scaleY in the serialized schema — see §9.4
  // for how width/height map to the Fabric scale at load time. width/height is
  // the single source of truth for rendered size; nothing else encodes size.
  x: number; y: number;    // CENTER of the box in template units (unrotated)
  width: number; height: number;   // rendered box in template units (authoritative)
  rotationDeg: number;     // clockwise, about the box center (= (x,y))
  opacity: number;         // 0..1
  locked: boolean;
  visible: boolean;
  z: number;               // stacking order within the side
}

export interface ImageLayer extends BaseLayer {
  type: 'image';           // user-uploaded raster artwork only (stickers are StickerLayer)
  blobKey: string;         // IndexedDB key for the image blob (NOT base64 in LS)
  mimeType: string;
  // naturalWidth/Height are the source bitmap dimensions, kept ONLY to preserve
  // aspect ratio and compute the Fabric scale (width/naturalWidth) at load.
  // They never override width/height as the rendered size.
  naturalWidth: number; naturalHeight: number;
  keepAspect: boolean;     // default true
}

// Stickers are first-party bundled SVGs, NOT user uploads. They keep their own
// layer type so recolor is serializable (MED-2) and so the blob resolver can
// recognize the reserved `sticker:<id>` key and load markup from the bundle,
// never from IndexedDB (MED-3, §8.5).
export interface StickerLayer extends BaseLayer {
  type: 'sticker';
  blobKey: string;         // ALWAYS the reserved `sticker:<id>` scheme (§8.5) —
                           // resolved to bundled SVG markup, never an IDB key
  mimeType: 'image/svg+xml';
  // same role as on ImageLayer: aspect + Fabric-scale derivation only
  naturalWidth: number; naturalHeight: number;
  keepAspect: boolean;     // default true
  // OPTIONAL recolor. When set, the resolver sets this hex as `fill` on the
  // sticker SVG's `[data-recolor]` target(s) at LOAD, before rasterizing for
  // Fabric (§8.5). Absent => the sticker renders in its authored colors.
  fill?: string;           // hex (#rgb/#rrggbb), validated like other hex inputs
}

export interface TextLayer extends BaseLayer {
  type: 'text';
  text: string;
  fontFamily: string;      // from fonts.config
  fontSize: number;        // template units
  fill: string;            // hex
  fontWeight: 400 | 700;
  italic: boolean;
  underline: boolean;
  align: 'left' | 'center' | 'right';
  lineHeight: number;
}

export type DesignLayer = ImageLayer | StickerLayer | TextLayer;
```

### 7.2 Versioned saved-design format (`src/types/design.ts`)

```ts
export const DESIGN_SCHEMA_VERSION = 1 as const;

export interface SavedDesignMeta {        // stored in localStorage (small)
  id: string;                             // draft id (nanoid)
  schemaVersion: number;                  // for forward migration
  name: string;                           // user-editable or auto "Untitled design"
  productId: string;
  colorId: string;
  sizeId: string;
  materialId: string | null;
  printSides: PrintSide[];                // sides that actually have layers
  quantity: number;
  // denormalized price snapshot for the list view (recomputed on open)
  unitPrice: number;                      // 450
  createdAt: string;                      // ISO
  updatedAt: string;                      // ISO
  thumbnailBlobKey: string | null;        // IndexedDB key for a small preview
}

export interface SavedDesignBody {        // stored in localStorage keyed by id
  id: string;
  schemaVersion: number;
  layersBySide: Partial<Record<PrintSide, DesignLayer[]>>;
}

// image/sticker blobs + exported previews + thumbnails live in IndexedDB,
// referenced by blobKey/thumbnailBlobKey.
```

Rationale for the split: metadata + layer geometry are small JSON (safe in localStorage, ~5MB budget); **binary artwork is large** and belongs in IndexedDB (per the locked stack). A `draftIndex` array in localStorage lists all `SavedDesignMeta` for the `/saved` page. Migrations: a `migrateDesign(raw)` function switches on `schemaVersion`; v1 is identity. Unknown/newer versions open read-only with a warning toast rather than crashing.

---

## 8. Asset & Garment-Recoloring Strategy

**Problem:** no photographic garment assets exist (only the logo). CSS-filtering a photo to recolor looks fake and the brief forbids it. **Decision: fillable SVG garment templates per product per side.**

### 8.1 SVG garment templates
Each supported product/side has a hand-authored (or sourced, suitably-licensed) **vector garment mockup** stored in `src/assets/garments/<product>/<side>.svg`. Each SVG:
- Has a fixed `viewBox` (the template coordinate space all layer geometry references).
- Exposes a single body element carrying a scoped `data-recolor="body"` attribute (the value is `GarmentTemplate.colorTargetId`, §6.1) whose `fill` we set at runtime to the selected `GarmentColor.hex`. The element may also keep an `id="garment-body"` for authoring readability, but recolor **matches on the `data-recolor` attribute within the instance's wrapper ref, never on a document-global id** (see the duplicate-id invariant below).
- Includes separate shaded overlay paths (seams, folds, collar shadow) painted with **semi-transparent black/white** (`multiply`/`screen` blend) that sit *above* the fill, so a single colored body reads as a realistically shaded garment across all colors — no per-color art needed. Collars/cuffs can have their own fill target (`data-recolor="trim"`) for two-tone products (e.g. polo).
- A subtle fabric texture is an optional low-opacity overlay.

This gives **instant, realistic recoloring for all ten colors from one asset per side**, scales crisply, is tiny, and never produces broken image URLs.

**Duplicate-id invariant (resolves NIT-2).** The category grid, hero, and showcase inline the same garment SVG many times in one document, which would make any internal `id` non-unique (invalid HTML) and would cross-wire any `url(#id)` reference (gradient/filter/clipPath) to the first match. The committed invariant for every garment SVG (and the §8.6 reference template): **it contains NO internal `url(#…)` references** (no `<linearGradient>/<filter>/<clipPath>` wired by id — the shading is flat semi-transparent overlays per §8.6, which needs none), and **recolor targets are matched by the scoped `data-recolor` attribute queried within each instance's own wrapper ref, not by a document id**. Any `id` left on the markup is decorative only and never queried or referenced. This keeps multiple inlined instances valid and independent with no id uniquification pass required.

**Inlining mechanism (concrete, no new dependency).** Runtime recolor (`wrapperRef.querySelector('[data-recolor="body"]').setAttribute('fill', …)`, scoped to the instance) and export serialization both require the garment SVG to be **inlined into the live DOM**, not referenced via `<img>` (an `<img src>` SVG is opaque to `querySelector` and recolor). The chosen mechanism is **Vite's `?raw` import** — `import bodySvg from '@/assets/garments/classic-tshirt/front.svg?raw'` yields the SVG markup as a string, which a small `<GarmentSvg>` component injects with `dangerouslySetInnerHTML` into a wrapper `<div role="img" aria-label="…">`. The source SVGs are trusted first-party build assets (not user input), so this injection carries no XSS surface. `GarmentTemplate.svg` (§6.1) holds the import key/path resolved to this raw string. After injection the component queries `[data-recolor="body"]` (and `[data-recolor="trim"]`) within its own wrapper ref — scoped to the instance, never a document-global id (NIT-2 invariant above) — and applies the color fill; the same inlined node is serialized on export so the runtime fill is captured. `vite-plugin-svgr` is explicitly **not** used (it would add a dep and turn SVGs into React components, complicating the raw-string serialization the exporter needs). This keeps the stack in §2 unchanged.

### 8.2 Layering model in the preview/canvas
Rendering order on the stage: (1) garment SVG (recolored) as the backdrop, (2) printable-area boundary guide (toggle-able), (3) the Fabric canvas holding the user's layers. **The live editing canvas does NOT hard-clip layers to the print area** — layers are placement-only (inserted centered in the area, but freely draggable past it). Overflow is explicitly allowed and surfaced as a non-blocking warning per §9.6; we never silently clip the editor. The garment SVG is rendered as a React element behind the Fabric `<canvas>`; the Fabric canvas is transparent so the garment shows through. The **only** place a print-area crop is applied is the optional **artwork-only PNG export** (§11.5), which crops to the print-area bounds purely for that output file; it never alters the live layers. Layer geometry is stored in template-viewBox units and mapped to canvas px via the current stage scale (§9.4), guaranteeing WYSIWYG across zoom, device, and export.

### 8.3 Optional raster upgrade path
`GarmentColor.images[side]` and a future `Product` raster field let the business drop in **real photos** later without code changes: if a dedicated image exists for the product+color+side, the renderer prefers it over the SVG. Until then SVG is the default. This satisfies "use real assets first; graceful configurable fallback."

### 8.4 Logo & favicon
Copy `CREAZYCARTLOGO.png` (never modify the original) into `src/assets/logo/` / `public/assets/logo/` as `logo-gold.png`. This single gold asset covers **all** placements via the §4.1 surface rule: directly on purple/dark surfaces, and inside a `--purple-800` rounded chip on the light header. A recolored purple/ink wordmark (`logo-ink.svg`) is an **optional** drop-in the header config can switch to later; it is NOT produced during this build (cleanly recoloring the raster to solid purple needs manual redraw/masking). **Favicons** (`favicon.ico` + `16/32/180/512` PNGs) are **pre-generated manually from the logo and committed to `public/`** — there is no favicon-generation tool in the stack (§2), so "at build" tooling is explicitly not assumed. Header picks the placement variant by surface via a `logoVariant` prop. All logo usages have descriptive `alt="CrazyCart.in — Shaping Ideas with Design"`. (Note: the raster wordmark reads "CrazzyCart.in"; brand text in copy uses the official **CrazyCart.in** spelling — see §16.)

### 8.5 Sticker library (local, editable)
A curated **starter set of local SVG vector stickers** in `src/assets/stickers/`, catalogued in `data/stickers.ts` (`{id,name,category,svg}`). Required starter categories/items: **stars, hearts, smiley faces, abstract shapes, flames, lightning bolts, flowers, geometric graphics, minimal decorative symbols** (at least ~3–5 per category). The "Add Sticker" tool opens a `StickerPicker`; selecting one **actually inserts it into the Fabric canvas as a fully editable/transformable `StickerLayer` with `type:'sticker'`** (move/resize/rotate/recolor-where-the-SVG-allows/duplicate/delete/layer-order) — **no dead buttons**. All stickers are original/brand-safe; no third-party or copyrighted clip-art.

**Recolor is persistable (resolves MED-2).** `StickerLayer` carries an optional `fill?: string` (§7.1). Each recolorable sticker SVG tags its recolor-eligible path(s) with a scoped `data-recolor` attribute (the same scoped-attribute convention §8.1/NIT-2 uses for garments, so no document-id collisions when a sticker is inlined more than once). The apply-at-load rule: the resolver parses the sticker's bundled SVG markup, and if `fill` is set, sets that hex as the `fill` on every `[data-recolor]` element **before** rasterizing the SVG to the image Fabric renders. A sticker with no `[data-recolor]` target simply ignores `fill` (the recolor control is hidden for it — still no dead button). Because the source stays **parsed/inline SVG markup (never a flattened bitmap)**, the recolor re-applies cleanly on every reopen/side-switch and the `fill` hex is all that needs to serialize.

**Storage — reserved `sticker:<id>` scheme only (resolves MED-3).** A `StickerLayer.blobKey` is **always** the reserved `sticker:<id>` form; it is **never** promoted into IndexedDB. The blob resolver recognizes the `sticker:` prefix and loads the bundled sticker markup from `data/stickers.ts` / `src/assets/stickers` directly (applying `fill` per the rule above), so bundled stickers **always** resolve even after a data-clear, IDB wipe, or private-mode fallback (§11.3) — they can never degrade to the missing-blob placeholder. IndexedDB promotion is reserved exclusively for **user-uploaded images** (`ImageLayer`). There is no "or" alternative.

### 8.6 Reference garment template & inventory (de-risks art production)

This removes the "hand-authored or sourced" ambiguity. **Every garment SVG is cloned and adapted from the single reference template below** — the implementer draws the body outline per garment/side but keeps the exact element structure (`data-recolor="body"` recolor target, optional `data-recolor="trim"` — matched by scoped attribute, not a document id, per the §8.1 NIT-2 invariant), the multiply/screen overlay convention, and the minimum visual bar. **What is NOT shared is the `viewBox`:** each garment/side declares its own `viewBox` sized to that silhouette's natural aspect ratio, because a cap, mug, sleeve or tote is not the 5:6 portrait of a tee and forcing them into one box would distort or empty the preview (hurting realism criterion 12). The reference below ships at `viewBox="0 0 1000 1200"` (a tee's 5:6 portrait) only as the structural exemplar — clone its ids/overlays/visual bar, not its dimensions. Recommended per-silhouette viewBoxes (all 1000 wide for consistent horizontal units, height set to the natural aspect):

| Silhouette | `viewBox` | Aspect |
|---|---|---|
| Tee / oversized / polo / sweatshirt / hoodie (front & back) | `0 0 1000 1200` | 5:6 portrait |
| Sleeve (left / right) | `0 0 1000 520` | wide strip |
| Cap (front) | `0 0 1000 620` | wide |
| Tote (front & back) | `0 0 1000 1100` | near-square-tall |
| Mug (front wrap panel) | `0 0 1000 780` | landscape-ish |

The studio stage derives its render aspect **from each template's own `viewBox`** (§9.4 stage scale `s = canvasPxWidth / viewBox.width`; the stage element's `aspect-ratio` is `viewBox.width / viewBox.height`), and every `PrintArea`'s `x/y/width/height` for a side is expressed in **that same template's viewBox units** (§6.1). `GarmentTemplate.viewBox` in the type model (§6.1) is therefore authoritative per side — the implementer sets it per SVG; nothing hard-codes 1000×1200. No external art is required and no per-color art is ever produced.

**Minimum acceptable visual bar (the definition of "realistic" for criterion 12):**
- One solid-fill body shape (`data-recolor="body"`, recolored at runtime), **not** a photo.
- 2–3 soft shading overlays (collar/neck shadow, sleeve/side fold, bottom hem) as semi-transparent black on `mix-blend-mode:multiply` and 1 optional highlight as semi-transparent white on `screen`, so the single filled body reads as a shaded garment across all 10 colors.
- Clean vector that stays crisp when rasterized to the 2000px export (§11.5); recognizable garment silhouette; collar/cuffs optionally a second fill target (`data-recolor="trim"`) for two-tone products (polo).
- No gradients keyed to a specific color, no baked-in color, no text. That is the whole bar — flat body + a few soft overlays. Anything above it is a bonus, not required to pass.

**Reference template (`front.svg`, tee exemplar at `viewBox="0 0 1000 1200"`); clone its id/overlay structure per garment/side but set each one's own `viewBox` from the table above:**
```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 1000 1200" role="img"
     aria-label="garment preview">
  <!-- 1) BODY: single recolor target, matched by the SCOPED data-recolor
          attribute (NOT the id — see §8.1 duplicate-id invariant NIT-2). fill
          is overwritten at runtime with the selected GarmentColor.hex. Keep
          this the only solid fill. The id is decorative/authoring-only. -->
  <path id="garment-body" data-recolor="body" fill="#cccccc"
        d="M300,150 L420,110 Q500,150 580,110 L700,150
           L780,320 L690,360 L690,1050 L310,1050 L310,360 L220,320 Z"/>
  <!-- optional 2nd target for two-tone trims (polo collar/cuffs); omit if unused -->
  <path id="garment-trim" data-recolor="trim" fill="#bbbbbb" opacity="0"
        d="M420,110 Q500,170 580,110 L560,180 Q500,215 440,180 Z"/>
  <!-- 2) SHADING overlays: color-agnostic, sit ABOVE the fill -->
  <g style="mix-blend-mode:multiply">
    <path fill="#000" opacity="0.14" d="M420,110 Q500,170 580,110 L560,200 Q500,235 440,200 Z"/>  <!-- neck/collar shadow -->
    <path fill="#000" opacity="0.10" d="M300,150 L220,320 L310,360 L330,210 Z"/>                   <!-- left sleeve fold -->
    <path fill="#000" opacity="0.10" d="M700,150 L780,320 L690,360 L670,210 Z"/>                   <!-- right sleeve fold -->
    <path fill="#000" opacity="0.08" d="M310,980 L690,980 L690,1050 L310,1050 Z"/>                 <!-- hem shadow -->
  </g>
  <g style="mix-blend-mode:screen">
    <path fill="#fff" opacity="0.06" d="M470,370 L530,370 L520,980 L480,980 Z"/>                   <!-- center highlight -->
  </g>
  <!-- 3) PRINT-AREA guide is NOT in the SVG; it is drawn by the studio from
          Product.printAreas (§9.6) so it stays toggle-able and side-specific. -->
</svg>
```
Runtime recolor (scoped to the instance wrapper ref, per the §8.1 invariant): `wrapperEl.querySelector('[data-recolor="body"]').setAttribute('fill', color.hex)` (and `[data-recolor="trim"]` for two-tone, toggling its `opacity`). The query is attribute-based, never a document-global id, so inlining the same SVG many times stays valid. Because overlays are color-agnostic, this one asset covers all 10 colors.

**Inventory scoped from §6.4 `supportedSides` (what the implementer must draw):**

| Product | Sides (SVGs) | Count |
|---|---|---|
| `classic-tshirt` | front, back, left-sleeve, right-sleeve | 4 |
| `oversized-tee` | front, back | 2 |
| `polo-tee` | front, back, left-sleeve | 3 |
| `crew-sweatshirt` | front, back | 2 |
| `pullover-hoodie` | front, back | 2 |
| `classic-cap` | front | 1 |
| `canvas-tote` | front, back | 2 |
| `custom-merch-mug` | front | 1 |

**Total: 17 garment SVGs**, all cloned from the one reference template's id/overlay structure (sleeve/cap/tote/mug are simplified silhouettes using the same id/overlay convention but their **own** viewBoxes per the aspect table above; back views reuse the front body's viewBox minus the collar notch). This is the full, countable art scope — no open-ended "sourcing" remains.

---

## 9. Design Studio Architecture

The studio is the core feature and **must be genuinely functional**. It is built around a thin Fabric.js abstraction so the rest of the app never imports Fabric directly.

### 9.1 Fabric commitment & isolation
**Fabric.js v6 is committed** (richer built-in object model, free-transform controls, text editing, serialization, and image filters out of the box — a better fit than Konva for an editor with text + transform handles + export). All Fabric calls are wrapped in `lib/canvas/CanvasEngine.ts` exposing a framework-neutral API:
```
createEngine(canvasEl, { printArea, onSelection, onChange }): CanvasEngine
engine.setSide(side) / addImage(blob, opts) / addText(opts) / addSticker(key)
engine.updateSelected(patch) / remove(id) / duplicate(id) / setLock(id,bool) / setVisible(id,bool)
engine.reorder(id, dir) / selectById(id) / clearSelection()
engine.setZoom(z) / fitToStage() / toArtworkPNG() / toMockupPNG(garmentSvgEl)
engine.serializeSide(): DesignLayer[]  /  engine.loadSide(layers, blobResolver)
```
React components talk only to `CanvasEngine` + the studio store. If Fabric ever needs replacing, only this file changes. The engine's public types (`CanvasEngine`, `EngineOptions`, `SelectionInfo`, `AddImageOpts`, `AddTextOpts`, `BlobResolver`) and the `FontDef` config type (§10.2) live in `src/types/editor.ts`; `lib/canvas/*` is the only place that imports `fabric`.

### 9.2 Per-side artwork preservation
The engine keeps **one serialized layer set per `PrintSide`** in the studio store (`layersBySide`). Switching sides: `serializeSide()` the current side into the store, clear the Fabric canvas, swap the garment template + print-area, then `loadSide()` the target side's layers (resolving `blobKey`→blob from IndexedDB/in-memory cache). Thus each side's artwork is fully preserved when the user flips front/back/sleeves. Only sides with ≥1 layer are reported as `printSides` in the saved meta and WhatsApp summary.

### 9.3 State management
Three React contexts, each a `useReducer` store:
- **StudioStore**: `activeProduct, colorId, sizeId, materialId, activeSide, quantity, layersBySide, selectedLayerId, zoom, showPrintArea, isDirty`, plus undo/redo stacks. Actions are pure and serializable; the reducer is the single source of truth and the Fabric engine is driven *from* it (store → engine) for structural changes, while live drag/resize events flow engine → store (debounced commit) to feed undo/redo.
- **SelectionProvider**: lightweight "current design in progress" surfaced in the header indicator and used to deep-link `/studio`.
- **SavedDesignsProvider**: wraps the storage repo (§11) for `/saved`.

### 9.4 Coordinate system & zoom
Template viewBox units are canonical. Stage scale `s = canvasPxWidth / viewBox.width`. Fabric objects are positioned at `unit*s`. Zoom multiplies `s` within `[0.5, 3]` with pan; `fitToStage()` resets. Export (§11.3) renders at a fixed high scale independent of on-screen zoom so output quality is stable.

**Authoritative size → Fabric scale mapping (resolves the one-way geometry rule).** The serialized schema stores only `x, y, width, height, rotationDeg` (template units) per layer; it does **not** store Fabric's `scaleX/scaleY`. The engine derives everything from `width/height` so save→reload→export can never disagree about size:

**Single pivot — Fabric CENTER origin (resolves HIGH-1).** Every Fabric object is created and read with `originX:'center', originY:'center'`, and the serialized `(x,y)` is the box **center** in template units (§7.1). Center origin is the only pivot in the system, so `rotationDeg` rotates about `(x,y)` identically whether the layer lives in the store or on the canvas, and rotated layers round-trip exactly on reopen and side-switch (criterion 13). No top-left is ever stored; any top-left a consumer needs is derived as `(x - width/2, y - height/2)` for the unrotated box, and bounds/overflow/crop use the rotated AABB (§9.6/§11.5) computed from this one definition.

- **Load (`loadSide`):** place the Fabric object with center origin, then set its on-screen scale from the authoritative box. For an image/sticker whose source bitmap is `naturalWidth × naturalHeight`, compute `fabricScaleX = (width * s) / naturalWidth` and `fabricScaleY = (height * s) / naturalHeight` (with `keepAspect:true`, `width/height` already preserve the natural aspect, so the two scales are equal). For a Fabric `Textbox`, set `fontSize = fontSizeUnits * s` and `width = width * s` (wrapping width); text height follows from wrapping and is read back on commit. Position: `left = x * s`, `top = y * s` (these are the object's **center** on canvas because origin is center), `angle = rotationDeg`.
- **Commit (engine → store):** on pointer-up / transform-end, **first normalize any transient scale into the authoritative metrics, then read geometry back.** For a `Textbox`, fold corner-scaling into the metrics (see MED-1 below) so no `scaleX/scaleY` leaks into the read; for an image/sticker read `width = obj.getScaledWidth() / s`, `height = obj.getScaledHeight() / s`. Then `x = obj.left / s`, `y = obj.top / s` (center, since origin is center), `rotationDeg = obj.angle`. Fabric's transient `scaleX/scaleY` are consumed here and discarded — they are never serialized.
- **Zoom** changes `s` only; it re-derives Fabric scale from the unchanged authoritative `width/height`, so zooming is lossless and never mutates stored geometry.
- **Export** uses a fixed export scale `sₑ` (defined in §11.5) in exactly the same formulas and the same center origin, guaranteeing the exported PNG matches on-screen placement (WYSIWYG, criterion 13).

**Text scale normalization (resolves MED-1).** A Fabric `Textbox` corner handle changes `scaleX/scaleY`, not `fontSize`, so a corner-scaled text layer would otherwise commit its pre-scale `fontSize` while rendering `fontSize*scaleY` tall and reload at the wrong size. The committed rule: on `scaling`/`modified` for a `Textbox`, set `fontSize = round(obj.fontSize * obj.scaleY)` and `width = obj.width * obj.scaleX`, then reset `obj.scaleX = obj.scaleY = 1` **before** reading geometry back; the store then takes `fontSizeUnits = obj.fontSize / s` and `width = obj.width / s` with no scale component surviving. (Side handles already change `width` only — the wrapping width — and need no fontSize change; the fontSize field in the Properties panel edits `fontSize` directly.) This keeps `width/height`/`fontSize` the single source of truth for text size across reload, side-switch and export.

This single-model rule (center origin everywhere, width/height/fontSize authoritative, Fabric scale always computed, scaleX/scaleY normalized away and never stored) is what makes side-switch preservation (§9.2) and reopen exact.

### 9.5 Undo/redo
Command-stack of store snapshots (structural changes: add/remove/duplicate/reorder/lock/visibility, and *committed* transforms). Live pointer dragging commits one undo entry on pointer-up (not per mousemove). Stacks capped at 50 entries. Keyboard: Ctrl/Cmd+Z / Ctrl/Cmd+Shift+Z, wired with an a11y-safe global handler that ignores events while a text input/`<textarea>`/Fabric text-editing is focused.

### 9.6 Printable area enforcement
The current side's `PrintArea` is drawn as a subtle dashed boundary (toggle via "show/hide"). New layers are inserted centered within the area and scaled to fit. The user **may** drag a layer partially outside (explicit action) — when any layer exceeds the bounds, a non-blocking inline warning appears ("Part of your artwork is outside the print area") and that state is flagged in the review summary. We never silently clip; we warn.

**Overflow test (rotated AABB from the single geometry definition, HIGH-1).** Bounds detection works purely from the stored `(x,y,width,height,rotationDeg)` in template units — `(x,y)` is the box center (§7.1). The layer's four unrotated corners are `(x ± width/2, y ± height/2)`; each is rotated by `rotationDeg` about the center `(x,y)`, and their min/max give the layer's axis-aligned bounding box (AABB). The layer is "outside" iff that AABB is not fully contained in the `PrintArea` rectangle (`x, y, width, height` for the side, also template units). This same center-based rotated-AABB computation is reused by the artwork-only export crop (§11.5), so the editor warning and the export crop agree exactly. A clear persistent note states the on-screen mockup is a visual preview and final print depends on artwork quality/garment/production — **no print-accuracy claim**.

### 9.7 Error handling — studio operations

| Operation | Failure condition | Recoverable? | Caller receives | Logged |
|---|---|---|---|---|
| Add image (upload) | wrong type / too large / decode fail | Yes | inline field error + toast, layer not added | `console.warn` (dev) |
| Engine init | WebGL/canvas unavailable | Fatal to studio | Full-panel fallback: "Your browser can't open the editor" + guidance | `console.error` |
| Load draft blob | blobKey missing in IDB | Partial | layer rendered as placeholder box + toast "An image couldn't be loaded" | `console.warn` |
| Serialize/export | canvas tainted / OOM | Yes | toast, export disabled, suggest smaller artwork | `console.error` |
| setSide | target template missing | Guarded by config validation | should never occur; dev throw, prod no-op + toast | `console.error` |

**Export blend-mode degradation (not a failure, a fidelity note):** if `mix-blend-mode` does not survive SVG→canvas rasterization on the target browser (iOS WebKit), the mockup export falls back to manual overlay compositing via `globalCompositeOperation` (§11.5). Placement/geometry are unaffected; worst case the mockup shading looks slightly flatter than the editor. No toast, not an error — handled silently by the committed fallback path.

---

## 10. Design Tools, Toolbar, Panels

### 10.1 Upload artwork
- Accept `image/png, image/jpeg, image/webp` (incl. transparent PNG). Reject others with a specific message.
- Validation (`utils/validation.ts` + `utils/imageProcessing.ts`): MIME **and** extension check; max file size **10 MB** (config constant); min dimension 50px; max dimension clamp 6000px (downscale on import to a working max of ~2000px longest edge to bound memory, preserving aspect). On any failure: no layer added, field-level error + toast listing the exact reason and accepted formats/limits.
- On success: store original blob in IndexedDB (`blobKey`), add an `ImageLayer` centered in the print area, `keepAspect:true`. Multiple images allowed. Guidance text recommends transparent PNG / high-resolution / print-ready artwork (informational, not a guarantee).

### 10.2 Text tool
Editable content, curated font list, font size, color (brand palette + free hex), bold, italic, underline, alignment, line-height, rotation, position/resize, duplicate, delete. Renders immediately via Fabric `Textbox`. Editing happens inline on canvas and/or in the properties panel.

**Input validation (text tool):**
- **Text content:** max **500 characters** (hard cap on input + paste, enforced in `utils/validation.ts`); empty text layer is allowed while editing but a layer left empty on blur is removed. No HTML — content is treated as plain text (Fabric `Textbox` renders strings, so no injection surface).
- **Free hex color input:** validated against `/^#([0-9a-f]{3}|[0-9a-f]{6})$/i`. On invalid input the field shows an inline error and **reverts to the last valid color** (never applies a bad value to the layer). The brand-palette swatches bypass validation (they are known-good tokens).
- **Font size / line-height / rotation:** clamped to sane ranges (fontSize 6–400 template units, lineHeight 0.8–3, rotation normalized to 0–359°).

**Font config (`data/fonts.ts`) — data-driven, typed in `src/types/editor.ts`:**
```ts
export interface FontDef {
  id: string;              // 'oswald'
  family: string;          // CSS font-family value used by Fabric + @font-face
  label: string;           // "Oswald" (shown in the picker)
  category: 'sans' | 'serif' | 'display' | 'script';
  weights: (400 | 700)[];  // weights shipped (bold toggle only enables 700 if present)
  files: Partial<Record<400 | 700, string>>; // woff2 path under public/fonts/
  fallback: string;        // CSS fallback stack if the file fails to load
}
export const studioFonts: FontDef[];   // curated set below
```
Starter set (all self-hosted woff2, brand-safe / open-licensed): **Inter, Fraunces, Montserrat, Poppins, Oswald, Bebas Neue, Pacifico, Lobster**. Adding/removing a studio font = editing `data/fonts.ts` + dropping its woff2 into `public/fonts/` (no code change).

**Font-loading guarantee (critical for correct rendering AND export).** Fabric rasterizes text with whatever glyphs the browser currently has; if a face is not yet loaded it silently falls back, and that wrong face would also bake into the exported PNG. To prevent this, selecting a font (or loading a draft whose text uses one) goes through a `loadFont(fontDef)` helper that constructs/awaits the `FontFace` (or waits on `document.fonts.load('<weight> 16px "<family>"')`), caches the loaded family, and only then applies it to the Fabric object and calls `canvas.requestRenderAll()`. Export (§11.5) `await`s `document.fonts.ready` plus every in-use face before rasterizing. While a font is loading the picker shows a tiny spinner on that item; on load failure the text keeps its previous face and a toast reports the specific font that couldn't load. Reduced-motion is unaffected (loading is not animation).

### 10.3 Toolbar (contextual)
Global: Add Image, Add Text, Add Sticker, Undo, Redo, Zoom (fit/in/out/reset), Toggle print-area, Preview (hide controls), Download Design, Save. Contextual (only when a layer is selected): Position (nudge/align), Scale, Rotate, Duplicate, Delete, Lock/Unlock, Hide/Show, Reorder (bring forward/back). Controls appear only when relevant to avoid clutter.

### 10.4 Layers panel
Lists layers of the **active side** top-to-bottom in z-order (images/stickers/text), shows type icon + name/thumbnail, current selection highlighted; actions: select, reorder (drag or arrows), hide/show, lock/unlock, delete. On mobile this is a collapsible bottom-sheet section.

### 10.5 Properties panel
Shows settings for the selected layer (text props or image props: size, rotation, opacity, aspect-lock, position), plus the **Design Summary** (product, color, size, material, active print sides, quantity, unit ₹450, total) and final actions (Save, Preview, Continue to Review).

---

## 11. Persistence, Export, Share

### 11.1 Storage layer (`services/designStorage.ts`)
- **IndexedDB** (via `idb`): one DB `crazycart`, stores: `blobs` (key→Blob for artwork), `thumbnails` (key→Blob), `exports` (cached preview PNGs, keyed `export:<draftId>`). Helper `putBlob/getBlob/deleteBlob`. All calls wrapped in try/catch; failures surface a toast and never crash. **`exports` cleanup policy (bounded):** keep at most **one export per draft** — writing a new export for a draft overwrites its prior `export:<draftId>` entry; deleting a draft deletes its export; and on app load any `export:*` whose `draftId` is no longer in `cc.draftIndex` is purged. The store therefore never grows beyond the number of saved drafts.
- **localStorage**: keys `cc.draftIndex` (`SavedDesignMeta[]`), `cc.design.<id>` (`SavedDesignBody`), `cc.prefs` (reduced-motion override, last product, **`dismissedAnnouncementId`** for the §5.2 announcement bar — a new announcement id re-shows the bar). A `StorageRepo` facade coordinates both (saving a design writes blobs to IDB first, then meta/body to LS; delete removes both). Quota-exceeded → toast asking the user to delete old drafts; partial writes are rolled back.

### 11.2 Save flow, auto-save & rename
**Manual save:** click Save → validate minimum (a product is selected) → serialize all sides → persist blobs then meta/body → generate a small thumbnail (mockup PNG at low scale) into IDB → **only after the storage write actually resolves** show "Design saved" toast and set `isDirty=false`. On failure show an error toast and keep `isDirty=true` (nothing is reported as saved that wasn't).

**Auto-save (where reliable):** a debounced (~2s after the last change) background save of the active draft, only when `isDirty` and a product is selected. Auto-save reuses the same repo path and the same success-gating — the "last saved" time updates only after the write resolves. Auto-save failures are silent except a subtle "Couldn't auto-save — changes kept in this session" indicator; they never fabricate a saved state and never block editing. Auto-save is skipped (with a one-time notice) if a prior quota error occurred, until the user frees space.

**Rename:** drafts can be renamed from the Saved Designs page and from the studio header (inline edit → updates `SavedDesignMeta.name` + `updatedAt`). The name is **trimmed and capped at 60 characters**; an empty/whitespace-only name falls back to "Untitled design".

Reopen from `/saved` via `?draft=<id>`; delete requires a confirm dialog (§11.5).

### 11.3 Storage error handling (concrete)

| Condition | Detection | Recoverable? | User-facing behavior | Logged |
|---|---|---|---|---|
| **Quota exceeded** | `QuotaExceededError` on LS/IDB write | Yes | Toast: "Storage is full — delete old drafts to save." Save reported as **failed**; `isDirty` stays true; auto-save pauses | `console.warn` |
| **Corrupted record** | JSON parse error / failed schema/`schemaVersion` check on load | Partial | Skip the bad draft, keep the rest; show it in `/saved` as "This draft is damaged and can't be opened" with a delete action | `console.error` |
| **Missing image blob** | `blobKey` not found in IDB on reopen | Partial | Design opens; the affected layer renders as a labeled placeholder; banner "Some images couldn't be recovered — re-upload to continue." Thumbnail shows a missing-image badge on `/saved` | `console.warn` |
| **IndexedDB unavailable** (private mode / blocked) | DB open rejects | Degraded | Fall back to metadata-only saving + session memory for blobs; warn once that drafts with images may not persist across reloads | `console.error` |
| **Newer `schemaVersion`** | version > current | Read-only | Open read-only with a warning toast rather than crashing (§7.2) | `console.warn` |

All artwork and metadata stay **local** (IndexedDB + localStorage); nothing is ever transmitted to an external service.

### 11.4 Saved Designs page (`/saved`)
Grid of draft cards, each showing: **garment mockup thumbnail** (from IDB, with a missing-image badge if the blob is gone), **product config summary** (product · color · size · material · print sides · qty · ₹450×qty), and **last-saved time** (relative, from `updatedAt`). Per-card actions: **Reopen** (→ `/studio?draft=<id>`), **Rename** (inline/dialog), **Delete** (always behind a **confirm-before-delete dialog**; removes both LS records and IDB blobs/thumbnail). A branded **empty state** ("No saved designs yet — start creating") links to `/studio`. Damaged or missing-blob drafts show the recovery messaging from §11.3 with a safe delete path.

### 11.5 Export / Download
Two explicit, clearly-labeled exports:
- **Artwork-only PNG** (transparent background, print-area-cropped, high scale) — labeled "Artwork preview (not a production-ready print file)". The crop rectangle is the current side's `PrintArea` (template units) scaled by `sₑ`; layer placement inside it is computed with the center-origin rotated AABB defined in §9.6/§7.1, so a rotated layer crops exactly where the editor warned it would.
- **Garment mockup PNG** (garment + artwork composited) — labeled "Mockup preview (shows placement only)".

**Export resolution (pinned, single-sourced — resolves NIT-1).** The export longest edge is **2000px**, declared once as the shared constant `EXPORT_LONGEST_EDGE = 2000` in `services/designExport.ts` (not duplicated per call). From it the fixed export scale is `sₑ = EXPORT_LONGEST_EDGE / viewBox.width` (every garment viewBox is 1000 wide per §8.6, so `sₑ = 2.0` in practice, but the formula stays viewBox-derived). **Both** the artwork-only and the mockup exports use this same `sₑ` and the same center-origin formulas from §9.4, so the two outputs and the on-screen stage stay WYSIWYG-consistent (criterion 13).

`services/designExport.ts` renders at this fixed `sₑ`; before rasterizing it **`await`s `document.fonts.ready` and every in-use `FontDef`** (§10.2) so text bakes in with the correct face, not a fallback. Mockup composites the garment SVG (rasterized to a canvas) under the Fabric export.

**Blend-mode fidelity on export (iOS WebKit caveat).** On-screen the garment uses inline DOM SVG, where `mix-blend-mode:multiply/screen` (the §8.6 shading overlays) renders reliably. When that SVG is rasterized to a canvas for the mockup export, in-SVG blend modes are historically unreliable in WebKit (iOS), so a naive rasterization could flatten the shading and make the exported mockup look duller than the editor. **Geometry and artwork placement — the actual §9.4 WYSIWYG guarantee (criterion 13) — are unaffected**, so this degrades gracefully. The implementer must verify blend fidelity on target browsers; **if unreliable, the exporter composites the overlays manually**: draw the recolored body first, then draw each shading overlay onto the export canvas with `ctx.globalCompositeOperation = 'multiply'` (shadows) / `'screen'` (highlight) instead of relying on the in-SVG blend. This manual path is the committed fallback and is noted here, in §9.7 (export row) and referenced from §8.6. Download via `URL.createObjectURL` + anchor click; also cached in IDB `exports` under `export:<draftId>` (one per draft, overwriting any prior export — see §11.1 cleanup policy) so the Review/WhatsApp step can offer the same file without re-rendering. **Never** labeled "print-ready".

### 11.6 Share — Review screen (`/studio/review`)
Dedicated final-review screen showing: product preview (mockup), selected product/color/size/material/print sides, text/artwork summary, quantity, unit price ₹450, total, a **Download preview** button, and the **Share on WhatsApp** button. "Back to editor" returns without losing the design (store persists; nothing cleared). Guard: if no active design, redirect to `/studio`.

---

## 12. Pricing (central config, `data/pricing.ts` + `utils/pricing.ts`)

```ts
export const PRICING = {
  currency: 'INR',
  symbol: '₹',
  baseDesignPrice: 450,          // per customized product — SINGLE SOURCE
  minQuantity: 1,
  maxQuantity: 999,
} as const;
export const unitPrice = () => PRICING.baseDesignPrice;
export const totalPrice = (qty: number) => PRICING.baseDesignPrice * qty;
export const formatINR = (n: number) => `₹${n.toLocaleString('en-IN')}`;
```
Shown consistently as **"Designs from ₹450"** on cards (+ "Create your own design"), **"Your custom design — ₹450"** in studio/summary, and in the WhatsApp message. Quantity is visible + configurable; **total = ₹450 × quantity**. The UI includes one honest disclaimer near totals: *"₹450 is the base design price. Production, bulk, shipping and taxes (if any) are confirmed by the CrazyCart team."* **No** invented sale prices, bulk discounts, shipping, tax, surcharges, fake reviews, or stock numbers anywhere.

**Quantity input validation (resolves NIT-3).** Quantity is adjusted via the `Stepper` (§4.5) or by typing directly. Both paths are **clamped to `[PRICING.minQuantity, PRICING.maxQuantity]` = `[1, 999]`**. For direct input: while typing, the field holds the raw string; on **blur** (or Enter), a **non-numeric or empty** value **reverts to the last valid quantity**, and an in-range numeric value is accepted; an out-of-range numeric value is **clamped** to the nearest bound (e.g. `0`→`1`, `1500`→`999`). The Stepper's − / + buttons disable at the respective bound. Totals (`totalPrice(qty)`) and every price display **recompute from the clamped value only** — the raw typed string never feeds a price. This mirrors the revert-on-invalid rigor of upload (§10.1), text (§10.2) and rename (§11.2).

---

## 13. WhatsApp Handoff

`services/whatsapp.ts` builds the message from **real, current selections** and URL-encodes it:

```
Hello CrazyCart Team!

I would like to order a customized product.

Product: <product.name>
Color: <color.name>
Size: <size.label>
Material: <material.name | "Not specified">
Print locations: <comma-joined active sides, e.g. "Front, Back">
Quantity: <qty>
Base price per item: ₹450
Estimated total: ₹<450*qty>

Design reference: <draft name> (ID: <draft id>)

Design details:
<per-side bullet summary of text content + "N image(s)" + any out-of-bounds note>

Please review my design and confirm the final printing details, availability, and total cost. I will attach the downloaded design preview manually in this chat.

Thank you!
```
The **Design reference** (local draft name + id) lets the team and customer refer to the same draft; it is a local identifier only (no server lookup). The message explicitly states the customer will **attach the downloaded preview manually**.

**Message length guard (prevents client truncation of the wa.me URL).** The per-side design-details block can balloon when text layers hold up to 500 chars across up to 4 sides. The builder therefore (a) truncates each **text layer's** contribution to the first **120 characters + "…"**, and (b) enforces an overall **soft limit of ~1500 characters** on the assembled (pre-encoding) message — if exceeded, the per-side text detail is collapsed to counts only ("Front: 2 text, 1 image") and a line is appended: "Full artwork is in the attached preview image." The structured header (product/color/size/material/sides/qty/price/reference) and the manual-attachment line are **never** truncated; only the free-text design detail is capped. The complete artwork always travels via the manually-attached preview image, so no information is lost.
URL: `https://wa.me/918208818421?text=<encodeURIComponent(message)>` opened via explicit user click (`window.open(url,'_blank','noopener')`).

**Pre-submit sequence (all required):**
1. Validate required selections (product, color, size; material required only if the product defines materials; at least one layer on at least one side). Missing → block with a clear message listing what's needed.
2. Confirm the user has saved/reviewed the design (we auto-save on reaching review).
3. Show a concise summary (the review screen itself).
4. Explain the preview image is **not auto-attached** — the user must attach the downloaded file manually in WhatsApp. This copy is shown next to the button.
5. Provide a direct **Download preview** action (the mockup PNG).
6. Preserve the design if the user returns (no state cleared).

We **never** claim WhatsApp received the order/image just because the link opened. The success copy reads "WhatsApp is opening — attach your downloaded preview and send to reach our team."

---

## 14. Responsive & Editor-Layout Strategy

### 14.1 Breakpoints (Tailwind)
`sm 640 · md 768 · lg 1024 · xl 1280 · 2xl 1536`. Marketing pages: single-column mobile → multi-column grids at `md`/`lg`. Container max 1240px.

### 14.2 Studio — desktop (≥ lg) 3-panel workspace
- **Left panel**: product config — color swatches, size selector, material selector (hidden if none), print-location tabs. Sticky.
- **Center**: large garment preview + interactive Fabric canvas + zoom controls + front/back/sleeve side switcher + print-area toggle. Takes remaining width; fits standard laptops with **no excessive horizontal scroll** (center column is flex-1, panels fixed-width ~300px/320px).
- **Right panel**: selected-layer properties, text/image settings, layers list, design summary, final actions. Sticky.

### 14.3 Studio — mobile (first-class, not a squeezed desktop)
Compact top bar (back + product name + save) → **large preview** with touch-friendly canvas → **horizontal scrollable** color/size selectors → **collapsible config drawers** → **bottom-sheet editing tools** (Add image/text/sticker, layer props, layers list via snap-point sheet) → **sticky bottom action area** with "Save Design" and prominent "Share on WhatsApp".
- While manipulating artwork, set `touch-action:none` + `overscroll-behavior:contain` on the canvas stage to **prevent accidental page scroll**, while normal page scroll works everywhere else.
- Keep controls/dialogs above the mobile keyboard and within safe areas (`env(safe-area-inset-*)`, `dvh` units). No clipped/overlapping/unreadable content; all touch targets ≥ 44px.

### 14.4 Shared layout rules
Header 72/64px; sticky scroll-aware. Images lazy-loaded (`loading="lazy"`, explicit width/height or aspect-ratio boxes) to **prevent layout shift**. Skeletons for catalog/preview loading. Empty states (no saved designs, no filter matches) are branded and actionable.

---

## 15. Animation, Accessibility, Performance

**Animation (Framer Motion):** page transitions (fade/slide 350ms), hero entrance, staggered product-card reveals (`whileInView`, 60ms stagger), subtle hero parallax, hover states, animated color-swatch selection, button press feedback, soft color transitions, editor panel/drawer/bottom-sheet/modal transitions, small save confirmation, scroll reveals, responsive loading/empty states. Animate **transform/opacity only**; no continuous/particle loops; no expensive scroll listeners (use `IntersectionObserver` + `requestAnimationFrame`-throttled scroll-direction hook). All gated by `usePrefersReducedMotion()` — reduced users get instant content, cross-fades at most, no parallax/translation. Motion never delays access to key content.

**Accessibility:** semantic HTML landmarks (`header/nav/main/footer`), one `h1` per page, logical heading order; all controls keyboard-operable with visible `--shadow-focus` ring; modals/drawers/bottom-sheets focus-trapped, Esc-closable, `role="dialog" aria-modal`, focus restored on close; color swatches are real buttons with `aria-pressed` + accessible names (not color-only); form fields labeled with error text wired via `aria-describedby`; canvas has a non-visual description + keyboard affordances for core ops (select/nudge/delete) and a documented limitation that fine canvas manipulation is pointer-oriented; icon-only buttons have `aria-label`; live regions announce save/validation toasts (`aria-live="polite"`). Target WCAG 2.1 AA for contrast and interaction. (Note: full WCAG conformance needs manual AT testing + expert review beyond automated checks.)

**Performance:** route-level code-splitting (`React.lazy`) — the studio + Fabric load only on `/studio`; lazy-load images; bound uploaded-image working size (§10.1); debounce canvas→store commits; memoize catalog filtering; keep bundles lean (tree-shake lucide, variable fonts subset). Target smooth interaction on mid-range Android. No console errors, no broken image URLs, no layout shift on image load.

---

## 16. Ambiguities Resolved (assumptions baked in)

1. **Logo spelling mismatch** — the raster reads "CrazzyCart.in" but the brand is **CrazyCart.in**. → All textual brand references use **CrazyCart.in**; the logo image is shown as-is (it's the supplied artwork). Recommend the owner supply a corrected logo later; swap is a single asset change.
2. **Canvas library** — brief allowed Fabric or Konva. → **Fabric.js v6, committed** (§9.1), isolated behind `CanvasEngine`.
3. **No garment photos exist** — → **Fillable SVG garment templates** with shaded overlays for realistic per-color recoloring (§8); raster upgrade path reserved in the type model.
4. **Display/body fonts unspecified** — → **Fraunces** (display) + **Inter** (sans), self-hosted woff2 with system fallbacks.
5. **Size measurements not provided** — → Size guide ships as a clearly-marked **placeholder** ("Measurements to be confirmed by CrazyCart") until the owner supplies real numbers; `SizeOption.chestInches/lengthInches` optional in the model.
6. **Materials per product** — → Material selector is **hidden** when a product defines no materials (model allows empty `materials`), so simple products (e.g. caps/totes) don't show an empty control.
7. **Max upload size & working resolution** — → **10 MB** limit, downscale imports to ~2000px longest edge for memory safety (both are config constants, easily changed).
8. **Which colors per product** — → Colors are per-product (`Product.colors`), defaulting to the full ten-color palette unless a product restricts it.
9. **Quantity bounds** — → min 1, max 999 (config); total strictly ₹450 × qty, no tiered pricing.
10. **Contact form without backend** — → Contact page uses WhatsApp + `mailto:` composition only; any visible "form" just builds a `mailto:`/wa.me link client-side. **No** network submit.
11. **Stickers source** — → Small bundled brand-safe SVG set; no third-party/copyrighted clip-art, avoiding licensing/legal risk.
12. **"Preview" vs "print-ready"** — → Exports are explicitly labeled **mockup** or **artwork preview**, never "print-ready"; a standing disclaimer states final print depends on artwork/garment/production.
13. **Category → studio linkage** — → Category/product "Customize" actions deep-link `/studio?product=&color=…`; catalog filters are URL query params so views are shareable/back-button-safe.
14. **Light vs dark logo** — → Two processed logo variants (gold on dark surfaces, recolored on light header) chosen per surface for contrast (§8.4).
15. **Out-of-print-area artwork** — → Allowed with an explicit non-blocking warning and a flag carried into the review summary; never silently clipped (§9.6).
16. **"Price" filter (brief said "price, where appropriate")** — → Every customized product shares the single ₹450 base design price, so a price range/sort filter would be meaningless and misleading. → **No price filter or price sort is shipped.** Catalog filters are category/color/material/style only; sort is featured / name A–Z / name Z–A / newest. If per-product pricing ever becomes real (owner-confirmed), the filter/sort engine already keys off config and a price facet can be added in `data/*` without component changes.
17. **Studio text-tool fonts vs site fonts** — → Site chrome uses Fraunces + Inter only; the text tool exposes a larger curated self-hosted set (§10.2) loaded lazily on `/studio`, with a `loadFont` guarantee so the correct face renders on canvas and in exports (never a silent fallback).

---

## 17. Acceptance Criteria (testable, numbered)

**Foundation & design system**
1. `npm install && npm run dev` starts the Vite SPA with no console errors; `npm run build` succeeds and `npm run lint` passes with zero warnings.
2. All colors/typography/spacing/radii/shadows/motion resolve from the token files; a grep finds no raw hex colors in `pages/`/`components/` (only in `styles/tokens` and `data/colors`).
3. Favicon and header logo render from the supplied logo asset; no broken image requests on any route.

**Navigation & pages**
4. Header shows logo + Home, Shop, Customize, How It Works, About, Contact + a prominent "Start Designing" button; it is sticky and scroll-aware; mobile shows a working slide-in drawer.
5. All routes in §5 render; unknown routes show the branded 404; navigating restores scroll to top.
6. Hero shows the exact locked headline/sub-copy and both CTAs; "Create Your Design" → `/studio`, "Explore Products" → `/catalog`.

**Catalog & product**
7. Shop shows all 8 category cards; clicking a category opens the filtered catalog or preselects it in the studio.
8. Catalog filters (category, color, material, style) are URL-synced and combine correctly; empty results show a branded empty state. (No price filter — see §16.16.)
9. Every product card shows "Designs from ₹450" and "Create your own design"; no sale prices, reviews, stock, or delivery claims appear anywhere.
10. Product detail shows large preview, color/size/material/print-location selectors, ₹450 price summary, care instructions when present, and a "Customize This Product" CTA that opens the studio preconfigured.

**Studio — core**
11. Selecting a product loads its config and renders the correct SVG garment; only that product's sizes/materials/sides are shown.
12. Selecting any of the ten colors instantly recolors the garment SVG realistically (filled body + shading), with the selected swatch clearly indicated (`aria-pressed`, ring, check).
13. Switching print side preserves each side's artwork exactly (positions, sizes, rotations, text) when switching back.
14. Uploading a PNG/JPG/JPEG/WebP (incl. transparent PNG) adds it to the garment; wrong type or >10 MB is rejected with a specific error and no layer added.
15. A placed layer can be moved, resized (aspect preserved by default), rotated, duplicated, deleted, locked/unlocked, hidden/shown, and reordered; multiple images are supported.
16. Text can be added and edited (content, font from curated list, size, color, bold/italic/underline, alignment, rotation, position, duplicate, delete) and renders immediately.
17. Undo/redo correctly reverse/replay the last ≥20 structural/transform operations; keyboard shortcuts work and are ignored while typing in inputs.
18. Zoom in/out/fit/reset work; "Preview" hides controls; print-area boundary toggles; dragging a layer out of bounds shows a non-blocking warning (never silently clipped).
19. The studio is usable with mouse, touch, and mobile pointer; on mobile the page does not scroll while manipulating artwork but scrolls normally elsewhere; controls stay above the keyboard and within safe areas; no clipped/overlapping content; touch targets ≥44px.

**Persistence / export / share**
20. Save stores artwork blobs in IndexedDB and metadata/geometry in localStorage; "Design saved" appears only after a successful write; large Base64 is never written to localStorage.
21. Saved drafts list on `/saved` with thumbnail + meta; reopening restores the full design (all sides/layers); delete removes both LS and IDB data after confirm.
22. Download produces a PNG labeled clearly as artwork-only or mockup; neither is labeled "print-ready".
23. Review screen shows product/color/size/material/print-sides/text-artwork summary/quantity/₹450 unit/total, a working Download, and "Back to editor" that returns without data loss.

**Pricing & WhatsApp**
24. Unit price ₹450 and total = ₹450 × quantity are consistent across cards, detail, studio, summary, and WhatsApp; the base-price disclaimer is present; no invented discounts/shipping/tax.
25. The WhatsApp button builds `https://wa.me/918208818421?text=…` with the exact locked template populated from real selections, opened only on explicit click.
26. Pre-submit, missing required selections block submission with a specific message; the UI explains the image must be attached manually and offers a direct preview download; nothing claims the order/image was received.

**Supplementary requirements**
27a. The app builds FLAT at `d:\CRAZYCART.IN` with no nested `crazycart/` folder; `CREAZYCARTLOGO.png` is copied (not moved) into `public/assets/logo/` and the original is unchanged; a `README.md` with install/dev/build + limitations exists.
27b. The Home page renders all 12 sections of §5.2 in order, including the announcement bar and FAQ accordion.
27c. The Shop page provides working search, category/color/style filters, sort, quick-preview modal, and a mobile filter drawer, with branded empty states and ₹450 on every card.
27d. The sticker picker contains the §8.5 starter categories and every sticker click inserts a real editable/transformable layer onto the canvas (no dead buttons).
27e. Auto-save persists the active draft after edits (success-gated), drafts can be renamed, and the Saved Designs page shows mockup thumbnail + config summary + last-saved time with reopen/rename/confirm-delete and missing-blob/corrupted-draft recovery messaging.
27f. Quota, corrupted-record, and missing-blob conditions each produce the §11.3 behavior without crashing; a save is confirmed only after the storage write resolves.
27g. The WhatsApp message includes the "Design reference" (draft name + id) line and the manual-attachment line.

**A11y / motion / perf**
28. With `prefers-reduced-motion: reduce`, no parallax/translation/looping animation plays and key content is immediately available.
29. All interactive controls are keyboard-reachable with a visible focus ring; modals/drawers trap focus, close on Esc, and restore focus; swatches/icon-buttons have accessible names.
30. The studio bundle (Fabric) is code-split and not downloaded on marketing routes; images lazy-load with no layout shift; no console errors in a full click-through.
31. Studio text renders and exports with the exact selected font (no silent fallback): applying a curated font and then exporting produces a PNG whose text uses that face.

---

## 18. Out of Scope

- Any backend/server, database, REST/GraphQL API, SSR.
- Payment gateway, checkout, order persistence, invoices.
- User accounts, login, profiles, server-side saved designs.
- Real-time order tracking, email/SMS sending from a server, admin dashboard.
- Automatic image attachment into WhatsApp (technically impossible via wa.me; handled by manual download + attach).
- Production-ready print-file generation (color profiles, bleed, DPI validation); exports are previews only.
- Real inventory, shipping rates, tax computation, bulk-discount logic (owner-confirmed, not invented).
- CMS; catalog is config-file-driven (editing config, not an admin UI, is the update path).
```

---

## 19. README.md Deliverable

Ship a `README.md` at the project root covering: project overview + brand; **prerequisites** (Node ≥18, npm); **install** (`npm install`), **dev** (`npm run dev`), **build** (`npm run build`), **preview** (`npm run preview`), **lint/test** (`npm run lint`, `npm run test`); where to edit catalog/pricing/colors/fonts/stickers (`src/data/*`); how garment recoloring works (fillable SVG templates) and how to drop in **real product photography later** via `GarmentColor.images[side]` with zero code changes; how local drafts/storage work (IndexedDB + localStorage, nothing transmitted); the WhatsApp handoff behavior incl. the **manual image-attachment** limitation; and an explicit **limitations note**: *no real product photography is bundled — SVG mockups are used as realistic, easily replaceable placeholders; size-guide measurements are placeholders pending real data; exports are previews, not production-ready print files.*

---

## 20. Review Responses (revision pass against `design-review.json`)

All 4 MEDIUM and 7 NIT findings were addressed; none were backlogged or ignored. Each resolution aligns with the locked stack and the original brief.

- **MEDIUM-1 (catalog never enumerated) — ADDRESSED.** Added §6.4: a concrete 8-product starter catalog (one per category) with explicit `id`, `categoryId`, `name`, `colors`, `sizes`, `materials`, `supportedSides`, `heroImageOrder`, `styleTags`, plus the matching `data/categories.ts` rows. This makes criteria 7/11 and §5.2.6 testable against real content and feeds §8.6's countable SVG inventory.
- **MEDIUM-2 (garment-SVG art hand-waved) — ADDRESSED.** Added §8.6: an embedded complete reference `front.svg` (fixed `viewBox="0 0 1000 1200"`, `id="garment-body"` recolor target, optional `garment-trim`, multiply/screen shading overlays), a stated minimum visual bar defining "realistic", runtime recolor snippet, and a per-product inventory totaling **17 SVGs** cloned from the one template. No open-ended "sourcing" remains.
- **MEDIUM-3 (dual size representation) — ADDRESSED.** §7.1 now drops `scaleX/scaleY` from the serialized schema; `width/height` in template-viewBox units is the single authoritative rendered size, `naturalWidth/Height` kept only for aspect. §9.4 documents the exact load/commit/zoom/export mapping (Fabric scale always computed as `width/naturalWidth`, never stored), guaranteeing WYSIWYG for criterion 13.
- **MEDIUM-4 (/shop vs /catalog) — ADDRESSED.** §3 now states explicitly there is **no** separate Catalog component; §5 route map marks `/catalog` as the "Shop (alias)" that mounts the same `<Shop/>` component with identical query-param handling. One component, two paths.
- **NIT-1 (type-scale tokens) — ADDRESSED.** §4.2 splits each step into separate `--fs-*/--lh-*/--ls-*` tokens and gives the Tailwind `fontSize` tuple form; the `size/lh` notation is flagged as doc shorthand.
- **NIT-2 (focus-ring width) — ADDRESSED.** Standardized on **3px** (`--shadow-focus`); the `--focus-ring` comment now points at it.
- **NIT-3 (raster logo recolor) — ADDRESSED.** §4.1/§8.4 adopt the gold-logo-on-`--purple-800`-chip default for the light header; a recolored vector is an optional later asset swap, not a build dependency.
- **NIT-4 (favicon tooling) — ADDRESSED.** §8.4 states favicons are pre-generated manually and committed to `public/`; no build-time favicon tool is assumed.
- **NIT-5 (unspecified validation) — ADDRESSED.** §10.2 adds text ≤500 chars + plain-text handling, free-hex regex `/^#([0-9a-f]{3}|[0-9a-f]{6})$/i` with revert-on-invalid, and clamped size/line-height/rotation ranges; §11.2 trims + caps rename at 60 chars.
- **NIT-6 (exports cleanup) — ADDRESSED.** §11.1/§11.5 define the policy: one export per draft (`export:<draftId>`, overwritten), deleted with its draft, and orphan `export:*` purged on app load — bounded to the number of saved drafts.
- **NIT-7 (loose color typing) — ADDRESSED.** §6.1 introduces a `GarmentColorId` union; `GarmentColor.id`, `Product.colors`, and `ProductCategory.previewColorId` now use it for compile-time safety alongside the dev-time validator (§6.3).

**Reviewer's unverified notes:** the truncated-brief concern (review note 1) is a limitation of the review input, not a design gap — the studio spec (print placement §6.1 `PrintArea`/§9.6, tools §10, export §11.5, WhatsApp §13) is fully specified here. The SVG recoloring realism (note 2) and canvas-taint on export (note 3) are now anchored by §8.6's concrete reference asset/visual bar and the §9.7/§11.3 handled failure cases (inline same-origin SVG, taint listed as a handled export failure).

### 20.1 Review responses — Pass 2 (against the current `design-review.json`)

Verdict was CHANGES_REQUESTED with 2 MEDIUM + 6 NIT findings. All 8 were **addressed**; none backlogged or ignored. Each resolution aligns with the locked stack (§2) and the original brief.

- **MEDIUM-1 (clip conflict: §8.2 "clipped" vs §9.6/§16.15 "never clip, warn") — ADDRESSED.** §8.2 rewritten: the live editing canvas does **not** hard-clip; layers are placement-only and overflow is allowed and warned per §9.6. The sole crop is scoped strictly to the optional artwork-only PNG export (§11.5), which never touches live layers. §9.6/§16.15 (criterion 18) now have no contradiction.
- **MEDIUM-2 (per-garment viewBox/aspect ambiguous) — ADDRESSED.** §8.6 now states each garment/side declares its **own** `viewBox` sized to its natural silhouette aspect, with an explicit recommended table (tee 1000×1200, sleeve 1000×520, cap 1000×620, tote 1000×1100, mug 1000×780). The reference SVG's 1000×1200 is flagged as the structural exemplar only (clone ids/overlays/visual bar, not dimensions). §9.4's stage scale and the stage `aspect-ratio` derive from each template's viewBox, and `PrintArea` bounds are confirmed to live in that same template's units. §6.1's `GarmentTemplate.viewBox` comment now says per-side, not a global 1000×1200.
- **NIT-1 (hero gold contrast) — ADDRESSED.** §5.1 binds the hero accent word to `--gold-600` (#a97e18, ~3.6:1 on cream, AA large) or a dark/purple chip; never `--gold-300/400` for text on light.
- **NIT-2 (announcement dismiss key undeclared) — ADDRESSED.** Stored in `cc.prefs.dismissedAnnouncementId` keyed by announcement id (new id re-shows the bar); enumerated in §11.1 and described in §5.2.
- **NIT-3 (header/nav listed as Home sections) — ADDRESSED.** §5.2 now splits the announcement bar + header into **layout-level chrome (L1/L2) rendered on every route**, and starts the Home module list at the editorial hero (item 3), eliminating the double-render risk on `/`.
- **NIT-4 (WhatsApp message length) — ADDRESSED.** §13 adds a per-text-layer cap (120 chars + "…"), an overall ~1500-char soft limit that collapses per-side detail to counts when exceeded, with the structured header + manual-attachment line never truncated; full artwork still travels via the attached image.
- **NIT-5 (mix-blend-mode on SVG→canvas export, iOS WebKit) — ADDRESSED.** §11.5 documents the caveat and the committed fallback (manual overlay compositing via `globalCompositeOperation` 'multiply'/'screen'); geometry/placement unaffected. Cross-referenced in §9.7 (fidelity note) and §8.6.
- **NIT-6 (SVG inlining loader not concrete) — ADDRESSED.** §8.1 commits to **Vite `?raw` import + `dangerouslySetInnerHTML`** (first-party trusted assets, no XSS surface, no new dep; `vite-plugin-svgr` explicitly rejected), so `[data-recolor]` is query-able at runtime and the runtime fill is captured on serialize. §6.1's `GarmentTemplate.svg` comment updated to reflect the raw-string mechanism. §2 stack unchanged.

### 20.2 Review responses — Pass 3 (against the current `design-review.json`)

Verdict was CHANGES_REQUESTED with 1 HIGH + 3 MEDIUM + 3 NIT findings. All 7 were **addressed** by applying each recommended fix verbatim; none backlogged or ignored. Each resolution aligns with the locked stack (§2) and the original brief. This is the final close-out pass — no further review gate follows.

- **HIGH-1 (rotation-origin contradiction breaks exact round-trip) — ADDRESSED.** Committed to a **single pivot: Fabric CENTER origin**, with the serialized `(x,y)` redefined as the box **center** in template units. §7.1 now documents `(x,y)` = center and rotation about `(x,y)`; §9.4 creates/reads every object with `originX/Y:'center'` and loads `left=x*s, top=y*s`, commits `x=obj.left/s, y=obj.top/s, rotationDeg=obj.angle`; §9.6 computes overflow from the **rotated AABB** of the four corners `(x±w/2, y±h/2)` rotated about the center; §11.5 crops the artwork-only export from that same rotated-AABB definition. One pivot, used identically everywhere, so rotated layers round-trip exactly (criterion 13). §7.1, §9.4, §9.6, §11.5 now agree.
- **MED-1 (Textbox corner-scaling lost on reload) — ADDRESSED.** Chose the **normalize-on-commit** option (over disabling corner handles) so users keep uniform corner scaling. §9.4 adds the rule: on `scaling`/`modified` for a `Textbox`, `fontSize=round(obj.fontSize*obj.scaleY)`, `width=obj.width*obj.scaleX`, then `obj.scaleX=obj.scaleY=1` **before** reading geometry back; the store reads `fontSizeUnits=obj.fontSize/s` with no scale surviving. Side handles still change wrapping width only; the fontSize field edits size directly. Text size now survives reload/side-switch/export.
- **MED-2 (sticker recolor has no field in the data model) — ADDRESSED, option (b).** Kept the advertised recolor feature (no dead buttons). §7.1 adds a dedicated **`StickerLayer`** (split out of `ImageLayer`) with optional **`fill?: string`**; `DesignLayer` is now `ImageLayer | StickerLayer | TextLayer`. §8.5 states the apply-at-load rule: the resolver sets `fill` on the sticker SVG's scoped `[data-recolor]` target(s) **before** rasterizing, and stickers are kept as **parsed/inline SVG markup, never a flattened bitmap**, so recolor re-applies on every reopen/side-switch and only the hex needs to serialize. Stickers with no recolor target hide the control.
- **MED-3 (sticker storage 'X or Y' with a correctness trap) — ADDRESSED.** Removed the "or". §8.5 commits to the reserved **`sticker:<id>` blobKey scheme** exclusively: the blob resolver recognizes the `sticker:` prefix and loads bundled markup from `data/stickers.ts` / `src/assets/stickers` directly, **never from IndexedDB**, so bundled stickers always resolve (even after data-clear / IDB wipe / private-mode fallback, §11.3). IndexedDB promotion is reserved for **user-uploaded images only**. §7.1's `StickerLayer.blobKey` is documented as always the `sticker:<id>` form.
- **NIT-1 (export resolution never pinned) — ADDRESSED.** §11.5 pins **export longest edge = 2000px**, single-sourced as `EXPORT_LONGEST_EDGE = 2000`, with `sₑ = EXPORT_LONGEST_EDGE / viewBox.width` used by **both** the artwork-only and mockup exports (and referenced from §9.4's export step).
- **NIT-2 (duplicate element ids on repeated inlining) — ADDRESSED.** Chose the **invariant** option: §8.1 states garment SVGs contain **no internal `url(#id)` references** and recolor targets are matched by a **scoped `data-recolor` attribute within each instance's wrapper ref**, not a document id. §6.1 (`colorTargetId` = the `data-recolor` value), §8.6 (reference SVG now carries `data-recolor="body"`/`"trim"`, runtime snippet queries the scoped attribute) updated to match. Any leftover `id` is decorative only.
- **NIT-3 (quantity direct-input unspecified) — ADDRESSED.** §12 adds the rule: quantity input is **clamped to [1,999]**; **non-numeric/empty reverts to the last valid value on blur**; totals **recompute from the clamped value** only. Stepper buttons disable at the bounds.