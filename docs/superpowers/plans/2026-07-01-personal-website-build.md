# Personal Website — Implementation Plan (Learning Roadmap)

> **Execution mode (overrides the standard plan format):** Jonathan writes 100% of the code; Claude teaches. Each task = **Learn** (Claude explains the concept map first) → **Build** (Jonathan writes it; deliverables + interfaces specified, implementations deliberately NOT provided) → **Verify** (exact commands/expected results) → **Checkpoint** (Claude reviews the code Socratically before the commit). Not for agentic execution — do not dispatch subagents to write this code.
> Spec: `Brain/docs/superpowers/specs/2026-07-01-personal-website-design.md`. After Task 0, copy this plan into the repo at `docs/superpowers/plans/` and check boxes there.

**Goal:** Jonathan's hand-drawn interactive desk-scene personal site, built entirely by Jonathan in Next.js, live on Vercel at Lighthouse ≥95.

**Architecture:** Static-first Next.js App Router site: every page a Server Component except one client island (`DeskScene`) driven by a single config file and one piece of state (`focusedObjectId`). Camera-zoom via Motion transforms on one wrapper div. Placeholder art first (plan C), real art swapped in by filename.

**Tech Stack:** Next.js 16.2 · React 19 · TypeScript strict · Tailwind 4.1 (CSS-first config) · Motion 12 (`motion/react`) · Playwright · Lighthouse CI · Vercel (default SSG — never `output: 'export'`)

## Global Constraints

- **Jonathan writes all code.** Claude reviews/teaches only; never edits the repo (vault planner wall also enforces from vault sessions).
- No `output: 'export'` — breaks `next/image` optimization (spec §Architecture).
- Animate **only `transform`/`opacity`**; zoom transform on one wrapper div; no permanent `will-change`.
- Art layers ship as WebP/AVIF **with alpha**, never raw PNG.
- Perf budget (CI-enforced): Lighthouse ≥95 all pages · scene LCP <2.5s simulated 4G · initial scene art <400 KB · no page-specific client JS on non-scene pages.
- Accessibility: all objects keyboard-operable; `<MotionConfig reducedMotion="user">`; Esc closes zoom.
- Package names: `motion` (import from `motion/react`), `tailwindcss` v4 (no `tailwind.config.js` — `@theme` in CSS).
- Commits: plain messages, no co-author trailer. PRs even solo once CI exists (Task 8+).

---

### Task 0: Repo, scaffold, first deploy

**Learn:** what `create-next-app` generates and why each file exists (`app/`, `layout.tsx` vs `page.tsx`, `globals.css`, `next.config.ts`, `tsconfig.json`); the dev → build → deploy loop; what Vercel does with a Next repo.

**Files:** entire scaffold at `~/Projects/my-desk/` (name = Jonathan's call; trivially renameable).

**Build (Jonathan):**
- [x] `npx create-next-app@latest my-desk` — TypeScript ✓, Tailwind ✓, App Router ✓, no `src/` dir (keep paths matching this plan), Turbopack default
- [x] Init GitHub repo `youjonathan/my-desk`, push
- [ ] Import into Vercel (defaults; no config changes)
- [ ] Copy this plan to `docs/superpowers/plans/2026-07-01-personal-website-build.md` in the repo

**Verify:**
- [ ] `npm run dev` → default page at `localhost:3000`
- [ ] `npm run build` → completes, all routes marked `○ (Static)`
- [ ] `https://my-desk-<hash>.vercel.app` serves the default page

**Checkpoint:** walk Claude through what each scaffold file does — in your own words, no notes. Commit.

### Task 1: Layout, fonts, Tailwind v4 theme

**Learn:** Root layout + Metadata API; `next/font` (why self-hosting kills layout shift); Tailwind v4's CSS-first config (`@import "tailwindcss"` + `@theme`) and how it differs from the v3 tutorials you'll find online.

**Files:** Modify `app/layout.tsx`, `app/globals.css`.

**Interfaces (produces):** CSS custom props via `@theme` (site palette + `--font-hand`, `--font-body`); layout exports `metadata` (title template `%s · Jonathan You`, description).

**Build (Jonathan):**
- [ ] Two fonts via `next/font/google`: a handwritten face for sticky notes (e.g. Caveat) + a body face; wire into `@theme` as font variables
- [ ] Site metadata; strip scaffold boilerplate to a placeholder home page

**Verify:**
- [ ] Both fonts render (test element per font); no `<link>` to fonts.googleapis.com in page source (`view-source` check)
- [ ] Tab title shows `Jonathan You`
- [ ] Checkpoint review → commit

### Task 2: Content pages — /resume, /projects, /blog, 404

**Learn:** Server Components (why these pages ship no page-specific JS); file-based routing; typed content modules as the no-CMS pattern; `notFound`/custom 404.

**Files:** Create `app/resume/page.tsx`, `app/projects/page.tsx`, `app/blog/page.tsx`, `app/not-found.tsx`, `content/projects.ts`, `components/ui/ProjectCard.tsx`.

**Interfaces (produces):** `content/projects.ts` exports `projects: Project[]` where `Project = { slug: string; name: string; oneLiner: string; stack: string[]; links: { label: string; href: string }[] }`. Consumed by `/projects` now, mobile landing (Task 7) later.

**Build (Jonathan):**
- [ ] `/resume`: resume + research content (source: `Personal/Resumes/Jonathan_You_Resume.tex` facts), print-friendly (`print:` variants)
- [ ] `/projects`: map `projects` → `ProjectCard` (start with 3–4: VHIL-E, Unleaded, Code2World-Web, claudes)
- [ ] `/blog`: coming-soon page, honest and charming
- [ ] `app/not-found.tsx`: placeholder for the crumpled-paper 404 (real art later)

**Verify:**
- [ ] All four routes render; `npm run build` keeps them `○ (Static)`
- [ ] Build output shows ~0 kB page-specific First Load JS deltas for these routes
- [ ] Checkpoint review → commit

### Task 3: The stage — scene config + positioned placeholder objects

**Learn:** the client/server boundary (`'use client'` — why exactly one island); designing a TS config schema so data, not code, drives a scene; fixed-aspect stage scaling (percentage positioning within an aspect-ratio box); `next/image` `fill` + `sizes`.

**Files:** Create `components/scene/scene.ts`, `components/scene/DeskScene.tsx`, `components/scene/SceneObject.tsx`, `public/scene/` (placeholder art). Modify `app/page.tsx` (render `DeskScene`).

**Interfaces (produces):**
```ts
type SceneObjectDef = {
  id: string;                       // 'monitor' | 'journal' | 'macbook' | ...
  name: string;
  art: string;                      // '/scene/journal.webp' — filename stable across placeholder→real swap
  x: number; y: number; w: number;  // % of stage (1600×1000 design space)
  tier: 'flavor' | 'portal';
  note: string;                     // sticky-note text
  noteSide: 'left' | 'right';
  href?: string;                    // portals only; external allowed (macbook)
};
export const sceneObjects: SceneObjectDef[];
export const STAGE = { w: 1600, h: 1000 };
```
All 10 spec objects present (monitor, journal, macbook, polaroids, calendar, binder, keychron, mouse, dock, fairy lights) with real note copy from the spec table.

**Build (Jonathan):**
- [ ] Placeholder art: labeled gray boxes or rough sketches as WebP-with-alpha (Claude supplies a one-liner `sharp`/`cwebp` command recipe; Jonathan runs it) + a base desk placeholder
- [ ] `DeskScene` ('use client'): aspect-ratio stage, base image (`priority`), maps `sceneObjects` → `SceneObject`
- [ ] `SceneObject`: positioned button-wrapped image, `aria-label={name}`, visible focus ring

**Verify:**
- [ ] All 10 objects positioned on the stage; layout survives window resize (proportions hold)
- [ ] Tab cycles through all 10 with visible focus, in a sane order
- [ ] Checkpoint review → commit

### Task 4: The zoom — camera, sticky notes, dismissal

**Learn:** React state as the single source of animation truth; Motion's `animate` prop + `AnimatePresence`; the camera math (given an object's rect + `noteSide`, derive the wrapper's `scale` + `translate` so the object frames off-center); interruptibility; `useReducedMotion`.

**Files:** Create `components/scene/StickyNote.tsx`, `components/scene/camera.ts` (pure function + its test). Modify `DeskScene.tsx`, `SceneObject.tsx`, `app/layout.tsx` (`<MotionConfig reducedMotion="user">`).

**Interfaces (produces):** `computeCamera(obj: SceneObjectDef, stage: {w,h}): { scale: number; x: number; y: number }` — pure, unit-testable. `DeskScene` owns `focusedObjectId: string | null`.

**Build (Jonathan) — TDD on the pure part:**
- [ ] **Failing test first** (`components/scene/camera.test.ts`, vitest or plain node runner — Jonathan picks with Claude): known object → expected scale/translate, object center lands off-center on the correct side. Run: expect FAIL (function missing)
- [ ] Implement `computeCamera` → test passes
- [ ] Wire into `DeskScene`: animated wrapper (`transform` only), click sets `focusedObjectId`
- [ ] `StickyNote` via `AnimatePresence` on the opposite side; portal notes get the "click here →" link
- [ ] Dismissal: Esc + click-outside + switching objects mid-animation (interruptible, no jank)
- [ ] Reduced-motion: crossfade instead of zoom

**Verify:**
- [ ] `npm test` green; every object zooms/returns smoothly; DevTools Performance shows no layout/paint storms during zoom (compositor-only)
- [ ] Keyboard-only run-through: Tab → Enter zooms → note readable → Esc returns
- [ ] macOS Reduce Motion on → crossfade
- [ ] Checkpoint review → commit

### Task 5: The monitor browser

**Learn:** DOM overlays aligned to image regions (screen rect as % of stage in config); nested interactive regions inside a zoomed scene; tab UI accessibility basics (`role="tablist"`, arrow keys optional).

**Files:** Create `components/scene/BrowserScreen.tsx`. Modify `scene.ts` (screen-rect constant), `DeskScene.tsx`.

**Interfaces:** `BrowserScreen` renders only when `focusedObjectId === 'monitor'`; home view = contact/links (email, GitHub, LinkedIn); tabs → `/resume`, `/projects` via `next/link`.

**Build (Jonathan):**
- [ ] Drawn-browser-styled overlay aligned over the monitor's screen area; default view = contact home; two tabs
- [ ] Links real and crawlable (`<a>`/`Link`, no onClick-only nav)

**Verify:**
- [ ] Zoom monitor → browser interactive; tabs navigate; alignment holds across viewport sizes
- [ ] Checkpoint review → commit

### Task 6: Mobile landing + responsive gate

**Learn:** CSS-first responsive gating vs JS detection (and why no UA sniffing); `next/dynamic` inside a client wrapper; what actually ships to a phone (network-tab audit as a skill).

**Files:** Create `components/MobileLanding.tsx`, `components/SceneGate.tsx` (client wrapper). Modify `app/page.tsx`.

**Interfaces:** `SceneGate` renders `DeskScene` ≥1024px, `MobileLanding` below; `MobileLanding` consumes `projects` from `content/projects.ts`. Desktop scene page includes a "prefer the simple version?" link.

**Build (Jonathan):**
- [ ] `MobileLanding`: name, one-liner, contact links, resume link, project list — designed properly, it's the recruiter page
- [ ] Gate: CSS-first (both in DOM, breakpoint-hidden) *or* matchMedia + dynamic import — Jonathan argues the tradeoff to Claude, then picks; requirement: phone downloads no scene art/Motion code

**Verify:**
- [ ] DevTools mobile emulation → landing page; network tab shows no scene chunks/art at 375px width
- [ ] Desktop → scene; "simple version" link works
- [ ] Checkpoint review → commit

### Task 7: Playwright smoke tests

**Learn:** e2e vs unit; Playwright projects (desktop + mobile viewport); resilient selectors (roles/labels, not CSS).

**Files:** Create `playwright.config.ts`, `e2e/site.spec.ts`.

**Build (Jonathan) — the suite (~6 tests):**
- [ ] Desktop: journal zoom → note visible → blog link navigates; monitor zoom → resume tab navigates; Esc returns camera; macbook link points at github.com/youjonathan
- [ ] Mobile viewport: landing renders, no scene; resume link works

**Verify:**
- [ ] `npx playwright test` green locally against `npm run build && npm start`
- [ ] Checkpoint review → commit

### Task 8: Lighthouse CI + perf hardening

**Learn:** lab vs field metrics; reading a Lighthouse trace (what LCP element is, render-blocking chains); GitHub Actions anatomy.

**Files:** Create `.github/workflows/ci.yml` (build + Playwright + `treosh/lighthouse-ci-action`), `lighthouserc.json` (assertions: perf ≥95, LCP <2500ms mobile-throttled, all routes).

**Build (Jonathan):**
- [ ] CI workflow: install → build → Playwright → LHCI against the built site
- [ ] Fix whatever fails until budget passes (expected suspects: base image `priority`/`sizes`, art payload, font loading)
- [ ] From here on: changes land via PR; CI gates merge

**Verify:**
- [ ] A PR shows both checks green; deliberately break the budget once (add a 2 MB image) → CI fails → revert (prove the gate is real)
- [ ] Checkpoint review → merge

### Task 9: Real art swap + launch pass

**Learn:** image-pipeline discipline end-to-end (export → alpha-WebP/AVIF → payload audit); the privacy pass as a release step.

**Files:** Replace files in `public/scene/`; Modify `scene.ts` positions to match real art; 404 art.

**Build (Jonathan):**
- [ ] Draw + export base desk scene and objects (iPad → WebP/alpha via the Task 3 recipe); swap by filename, tune config positions
- [ ] Scan polaroids; **privacy pass** — confirm each pictured friend is fine being on a public site; mask into drawn frames
- [ ] Crumpled-paper 404 art
- [ ] Sticky-note copy polish (voice pass — his words, not Claude's)

**Verify:**
- [ ] Initial scene art <400 KB (network tab, disk cache disabled); CI still green; Lighthouse ≥95 on the deployed URL
- [ ] Full keyboard + reduced-motion + mobile passes on production
- [ ] Vault debrief: log the launch, flag the card → **site is live; v2 shelf stays parked**

---

## Self-review notes (2026-07-01)

- Spec coverage: all spec sections map to tasks (scene/zoom → 3–4, browser → 5, mobile → 6, perf/a11y → 4/6/8, testing → 7, art pipeline → 3/9, 404 → 2/9). Blog pipeline + v2 shelf intentionally out of scope per spec.
- Deliberate format deviation: no full implementations, per execution mode at top. Interfaces/type shapes provided; bodies are Jonathan's.
- Type consistency: `SceneObjectDef`/`sceneObjects`/`focusedObjectId`/`computeCamera` names used consistently across Tasks 3–7.
