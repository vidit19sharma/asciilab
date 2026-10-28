# asciilab v1 — Implementation Plan

**Spec:** `docs/superpowers/specs/2026-10-28-asciilab-design.md` (the spec is normative; this
plan only decomposes it into executable tasks and fixes decisions the spec leaves open).

**Goal:** Ship asciilab v1.0.0 — the web app live on Cloudflare Pages, `asciilab` and
`@asciilab/engine` published on npm, every §8 Definition-of-Done item checked.

**Execution model:** One fresh subagent per task. Each task: read the spec sections listed, read
the files listed under *Reads*, write failing tests first, implement, run the package's
`test`/`typecheck`/`lint`, commit **only the paths it owns** with a Conventional Commit, report.
Tasks in the same wave run in parallel and touch disjoint paths. A task that needs something
outside its paths stops and reports instead of improvising.

**Tech stack (pinned at T0):** pnpm 10.x · TypeScript 5.9.x · Biome 2 · Vitest + fast-check ·
tsdown · Changesets 3 · Vite 7.x + React 19 + Tailwind v4 + Zustand 5 + zod/mini (zod 4) ·
Radix slider/select/tabs/dialog · Playwright + @axe-core/playwright · sharp · @napi-rs/canvas
(dev only) · Cloudflare Pages via wrangler.

---

## Ground rules for every task

1. **No new dependencies.** T0 installs every dependency for every package. If a task needs a
   package that is not installed, it stops and reports.
2. **Own your paths.** Commit with `git add <explicit paths>`; never `git add -A`. If `git commit`
   fails with `index.lock`, `sleep 3` and retry (max 5).
3. **Green means:** `pnpm --filter <pkg> test` passes, `pnpm --filter <pkg> typecheck` passes,
   `pnpm biome check <pkg-dir>` passes. Report the exact commands and their final lines.
4. **Engine API is a contract.** Names and signatures in spec §3.1, §3.3, §3.4, §3.5 are exact.
   A task that believes the spec is wrong adds a `SPEC NOTE:` line to its report; it does not
   silently change the API.
5. **No `any`, no `@ts-ignore`, no `as unknown as`.** `noUncheckedIndexedAccess` is on; use `!`
   only where the index is provably in range (Biome's `noNonNullAssertion` is disabled).
6. **Determinism.** No `Math.random` in the engine. Ties break to the lowest codepoint.
7. **Tests that can fail.** Every test must assert something that the implementation could get
   wrong. Snapshot/golden files are reviewed by eye before commit.

---

## Wave 0 — Foundation

### T0 · Monorepo scaffold and full dependency install

**Owns:** `/package.json`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`, `.npmrc`, `.node-version`,
`biome.json`, `tsconfig.base.json`, `tsconfig.json` (root, project references),
`.changeset/config.json`, `.changeset/README.md`, `.github/workflows/ci.yml`,
`apps/web/**`, `packages/engine/**`, `packages/cli/**` (stubs only), `README.md` (dev section).

**Spec:** §6, §7.4, §9.

**Deliverables**
- `npm i -g pnpm@10` documented; `packageManager` field pinned to the installed 10.x.
- Root scripts: `lint` (`biome check .`), `format` (`biome check --write .`), `typecheck`
  (`pnpm -r typecheck`), `test` (`pnpm -r test`), `build` (`pnpm -r build`), `changeset`,
  `release` (`pnpm -r build && changeset publish`).
- `biome.json`: recommended rules, 2-space, 100 cols, double quotes, semicolons, organize
  imports, `style/noNonNullAssertion: off`, `css.parser.tailwindDirectives: true`, ignore
  `dist`, `coverage`, `playwright-report`, `test-results`, `**/data/**/*.json`, `**/*.flf`.
- `tsconfig.base.json`: `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`,
  `verbatimModuleSyntax`, `isolatedDeclarations`, `module: preserve`, `moduleResolution:
  bundler`, `target: ES2023`, `lib: [ES2023]`, `noEmit`. Packages add `lib: [DOM]` where needed.
- `packages/engine`: name `@asciilab/engine`, `type: module`, `sideEffects: false`, exports
  `./dist/index.js` + types, `files: ["dist"]`, `publishConfig.access: public`,
  `repository.url`, scripts `build` (tsdown, ESM, dts, platform neutral, `fixedExtension:
  false`), `typecheck`, `test` (vitest run), `test:update-goldens` (`UPDATE_GOLDENS=1 vitest
  run`), `gen:atlas`, `gen:fonts`. devDeps: vitest, @vitest/coverage-v8, fast-check, tsdown,
  typescript, @napi-rs/canvas. `src/index.ts` exports `export const VERSION = "0.0.0"`;
  one smoke test.
- `packages/cli`: name `asciilab`, `bin: { asciilab: "./dist/cli.js" }`, deps
  `@asciilab/engine: workspace:*`, `sharp`; devDeps vitest, tsdown, typescript, @types/node.
  `src/cli.ts` prints `--version`; one spawn smoke test.
- `apps/web`: name `@asciilab/web`, `private: true`, Vite 7 + React 19 + Tailwind v4 hello
  world that builds; deps react, react-dom, zustand, zod, @radix-ui/react-{slider,select,tabs,
  dialog}, @asciilab/engine (workspace), @fontsource-variable/jetbrains-mono; devDeps vite,
  @vitejs/plugin-react, tailwindcss, @tailwindcss/vite, typescript, vitest, happy-dom,
  @playwright/test, @axe-core/playwright, pngjs, @types/pngjs, @types/node, @lhci/cli.
  Scripts `dev`, `build`, `preview`, `typecheck`, `test` (vitest), `test:e2e` (playwright).
- `.changeset/config.json`: `baseBranch: main`, `access: public`, `ignore: ["@asciilab/web"]`,
  `fixed: [["@asciilab/engine", "asciilab"]]`.
- `.github/workflows/ci.yml`: on PR + push main; `pnpm/action-setup@v4` (reads
  `packageManager`), `actions/setup-node@v4` Node 22 with pnpm cache; steps: install
  `--frozen-lockfile` → `pnpm lint` → `pnpm typecheck` → `pnpm test` → `pnpm build`.
  (Playwright and deploy steps are added by T2.9; CLI smoke by T3.1.)
- README "Development" section: install, scripts, package table.

**Done:** `pnpm install --frozen-lockfile && pnpm lint && pnpm typecheck && pnpm test && pnpm
build` all exit 0 from a clean clone. `pnpm ls -r --depth 0` shows every dependency above.
Commit `chore: scaffold pnpm monorepo, tooling, CI`.

---

## Wave 1 — Engine core types

### T1.1 · Types, cell buffer, option validation, geometry, sRGB

**Owns:** `packages/engine/src/{types,cellbuffer,options,geometry,srgb}.ts` + tests.
**Spec:** §2.1, §3.1, §3.2 rows *Geometry*; §9 defaults.

**Contract (exact):** every type and the `DEFAULT_OPTIONS` values of spec §3.1 and plan T1
preamble below. `createCellBuffer(cols, rows): CellBuffer`. `resolveOptions(partial):
RenderOptions` merges with defaults (explicit `undefined` does not override) and throws
`RangeError` whose message starts with the field name for every out-of-range value (ranges
in §3.1 comments; `cols` 8..400 is the **web** cap — the engine accepts 1..10000).
`subcellGrid(mode, blockSet): { sx, sy }` (tone/edge 1×1, braille 2×4, half 1×2, quadrant
2×2, sextant 2×3, glyph 4×8). `rowsFor(frame, cols, cellAspect): number` =
`max(1, round(cols * (h/w) * cellAspect))`. `srgbToLinear: Float32Array(256)` LUT and
`linearToSrgb(x: number): number` (0..255, rounded).

`DEFAULT_OPTIONS`: mode `tone`, cols 120, cellAspect 0.5, ramp `RAMPS.standard` string, invert
false, brightness 0, contrast 0, gamma 1, luma `rec709`, dither `none`, ditherStrength 1,
colorMode `none`, colorDepth `truecolor`, background `dark`, blockSet `quadrant`,
edgeThreshold 0.25, edgeBlend 0.6, glyphSet = printable ASCII 32–126 + `▀▄█▌▐░▒▓` +
`─│┌┐└┘├┤┬┴┼╱╲╳`, workFactor 0.5, seed 0.

**Tests:** defaults frozen and complete; each field's RangeError (one table-driven test);
`rowsFor` cases (e.g. 256×64 @ 120 cols, 0.5 → 15); subcell table; LUT monotonic and
`linearToSrgb(srgbToLinear[i]) === i` for all i.

**Done:** green; commit `feat(engine): core types, options, geometry, sRGB LUT`.

---

## Wave 2 — Engine stages (parallel) + web shell + FIGlet

### T1.2 · Sampling and tone

**Owns:** `packages/engine/src/{sample,tone}.ts` + tests. **Reads:** T1.1 files.
**Spec:** §3.2 rows *Resample*, *Tone* (as amended: no invert here).

**Contract:** `sampleFrame(frame, cols, rows, sx, sy, background, out?): SubcellField` where
`SubcellField = { w, h, rgb: Float32Array(w*h*3) /*linear, composited*/, alpha:
Float32Array(w*h) }`; area-average in linear light with fractional pixel coverage (works for
upsampling too); alpha composited against black/white for dark/light, kept straight for
transparent. `toneField(field, opts, out?): Float32Array` = luma (709/601) → contrast around
0.5 (`c` in -1..1 → factor `(1+c)/(1-c)` clamped) → brightness offset → clamp → `pow(1/gamma)`
→ clamp. `cellAlpha(field, sx, sy, out?)` mean alpha per cell.

**Tests:** 2×2 checker → 1 subcell = linear mean; alpha 0.5 red over white/black/transparent;
upsample 1×1 → 4×4 constant; tone of pure white = 1, black = 0, 50% sRGB gray ≈ 0.214;
contrast ±1 and gamma extremes stay in [0,1]; property: output length = w*h.

**Done:** green; commit `feat(engine): linear-light sampling and tone mapping`.

### T1.3 · Dithering and ramps

**Owns:** `packages/engine/src/{dither,ramps}.ts` + tests. **Reads:** T1.1 files.
**Spec:** §3.2 rows *Dither*, *Ramp* (as amended: `calibrateRamp` signature).

**Contract:** `quantize(values, w, h, levels, dither, strength, seed, out?): Uint16Array` →
level index 0..levels-1 per element. `none`: `round(v*(levels-1))`. Bayer 2/4/8: recursive
matrix, threshold normalized to ±0.5/levels scaled by strength. `noise`: seeded integer hash
(e.g. `hash(x,y,seed)` via multiply/xor-shift, document it). `floyd-steinberg` (7/16 3/16
5/16 1/16) and `atkinson` (6 × 1/8) on a scratch copy, scan order, error × strength.
`RAMPS` frozen record `{ standard, long70, blocks, minimal, binary }` with the exact strings in
§3.2. `calibrateRamp(glyphs, coverage)` per amended spec.

**Tests:** uniform 0.5 field with bayer4 → exactly 8/16 on; uniform 0.25 bayer2 pattern;
noise is deterministic for a seed and differs between seeds; FS of uniform 0.5 over 2 levels
→ ~50% on with no run longer than 2; strength 0 == `none`; `calibrateRamp(" @.", {32:0,
64:1, 46:0.1}) === " .@"`; drops unknown glyph; returns input if <2 survive.

**Done:** green; commit `feat(engine): dithering and character ramps`.

### T1.8 · FIGlet parser, renderer, fonts

**Owns:** `packages/engine/src/figlet/**`, `packages/engine/data/fonts/**`,
`packages/engine/scripts/gen-fonts.ts`, `packages/engine/test/figlet/**`. **Reads:** T1.1.
**Spec:** §3.4 (as amended), §7.1 FIGlet goldens.

**Contract:** `parseFlf(text): FigFont` per `figfont.txt` (signature `flf2a`, hardblank,
height, baseline, maxLength, oldLayout, commentLines, printDirection?, fullLayout?,
codetagCount?; chars 32–126, then 196 214 220 228 246 252 223, then code-tagged; endmark
stripping; oldLayout→fullLayout fallback). Throws `Error("Compressed .flf not supported")`
on `PK\x03\x04`. `renderFiglet(text, font, opts?): string[]` with the six horizontal smush
rules + universal smushing, `full`/`fitted`/`smushed`/`controlled`, word-wrap at `width`,
`align`. `figletToCells(lines): CellBuffer`. `FONTS: Readonly<Record<string, string>>` for the
10 fonts — generated by `scripts/gen-fonts.ts` into `src/figlet/fonts.generated.ts` as string
constants (no `?raw`; works in Node and bundlers). Fonts downloaded from
`https://github.com/cmatsuoka/figlet/tree/master/fonts` (standard, slant, small, big, banner,
block, shadow, script, mini, digital); header comments reviewed; `data/fonts/LICENSES.md`
records the status of each.

**Tests:** header parse of `standard.flf` (height 6, hardblank `$`); each smush rule with a
hand-built 2-char font; `renderFiglet("Hi", standard)` equals the known 6 lines; wrap at
width 20 breaks on a space; align center pads correctly. **Reference goldens:** for each
font, `figlet -f <font> -w 200 "Hello, World! 123"` captured on a machine with `brew install
figlet` into `test/figlet/goldens/<font>.txt` (trailing whitespace trimmed per line) and
compared to `renderFiglet(..., { layout: 'controlled', width: 200 })`. If `figlet` is not
installed the task installs it with Homebrew; if that fails, it marks those 10 tests
`.todo` and reports — it does not hand-write goldens.

**Done:** green; commit `feat(engine): FIGlet parser, renderer, bundled fonts`.

### T2.1 · Web shell, theme, routes, static pages, Playwright harness

**Owns:** `apps/web/**` except files owned by T2.2–T2.9 (listed in those tasks).
**Reads:** spec §4.1, §4.2, §4.4, §4.6, §4.7 (theme/focus), §4.9 (meta only), §4.10, §7.2.

**Deliverables**
- `index.html`: title, description, canonical (from `VITE_SITE_URL`, default
  `https://asciilab.pages.dev`), OG/Twitter meta pointing at `/og.png`, favicon links, font
  preload, `<noscript>`.
- `src/app/index.css`: Tailwind v4 `@import "tailwindcss"`, `@theme` tokens — dark: bg
  `#0b0b0c`, surface `#121214`, line `#232326`, line-strong `#3a3a3f`, text `#e8e8ea`, muted
  `#8b8b92`, accent `#f5b84a` (amber); light: bg `#fafafa`, surface `#ffffff`, line `#e4e4e7`,
  text `#161618`, muted `#5d5d66`, accent `#8a5a00`; `--font-mono` JetBrains Mono Variable
  stack; `--font-sans` system stack; radius 2px/4px; `[data-theme]` + `prefers-color-scheme`
  fallback; focus ring 2px accent; `prefers-reduced-motion` kills transitions. State the
  contrast ratios in a comment (all text ≥ 4.5:1, UI lines ≥ 3:1).
- `src/app/theme.ts`: `system|dark|light` in `localStorage("asciilab.theme")`, applies
  `data-theme`; `public/theme-init.js` (external, CSP-safe) applies it before paint.
- `src/app/router.ts`: `useRoute()` via `useSyncExternalStore` on `popstate`; `navigate(path)`;
  routes `/`, `/text`, `/privacy`, `/about`, else 404 view. Unit test for `matchRoute`.
- `src/app/App.tsx`, `Header.tsx` (wordmark, Radix Tabs Image/Text with `activationMode=
  "manual"`, PrivacyBadge → `/privacy`, theme toggle, `?` help button placeholder, GitHub
  link), `Footer.tsx` (GitHub, npm, privacy, about).
- `src/pages/Privacy.tsx`, `About.tsx` with the copy from spec §4.6 (incl. the URL-load
  caveat) and credits (FIGlet fonts, JetBrains Mono, chafa inspiration). `usePageMeta(title)`
  sets `document.title` and canonical per route.
- `src/strings.ts`: every user-visible string in the app so far; later tasks append.
- `public/_headers` with the exact CSP from amended §4.6 plus `X-Content-Type-Options:
  nosniff`, `Referrer-Policy: no-referrer`, `Permissions-Policy: camera=(), microphone=()`;
  `public/_redirects` (`/* /index.html 200`); `vite.config.ts` applies the same CSP to
  `preview` (parse `_headers`; unit test asserts they match) and sets `worker.format: "es"`.
- `src/app/analytics.ts`: injects the Cloudflare beacon **only** when `import.meta.env.PROD &&
  import.meta.env.VITE_CF_BEACON_TOKEN && import.meta.env.VITE_ANALYTICS !== "0"`.
- `playwright.config.ts`: projects chromium + webkit, `webServer: "VITE_ANALYTICS=0 pnpm build
  && pnpm preview --port 4173"`, baseURL, `reuseExistingServer: !CI`; `e2e/helpers.ts`
  (`waitForRender`, `dropFile`, `pickSelect`); `e2e/shell.spec.ts`: header renders, tabs
  navigate, `/privacy` and `/about` deep-link (preview SPA fallback), theme toggle sets
  `data-theme`, CSP header present on preview responses, console has no "Refused to" errors.

**Done:** `pnpm --filter @asciilab/web build && test && test:e2e -- shell` green; commit
`feat(web): app shell, theme, routing, static pages, Playwright harness`.

---

## Wave 3 — First end-to-end render

### T1.4 · Color, tone + edge modes, render orchestrator

**Owns:** `packages/engine/src/{color,render}.ts`, `src/modes/{tone,edge}.ts` + tests.
**Reads:** T1.1–T1.3. **Spec:** §3.2 rows *Mode: tone*, *Mode: edge*, *Color*,
*Orchestrator* (as amended: flip after quantization); §2.1.

**Contract:** `PALETTE16: Uint8Array(48)` (VGA), `PALETTE256` (16 + 6×6×6 cube + 24 grays),
`nearestIndex(palette, r, g, b): number` in linear RGB, ties → lowest index;
`finalizeColors(cells, field, grid, opts)` writes fg/bg per §3.2 *Color* and quantizes by
depth. `renderTone(ctx)` and `renderEdge(ctx)` where `ctx = { field, tone, cols, rows, grid,
opts, flip, cells, scratch }`. Edge: Sobel 3×3 on the per-cell tone (clamped borders),
magnitude normalized so a unit step = 1, orientation buckets at 0/45/90/135 ±22.5° → `-` `/`
`|` `\`, `_` when horizontal and the lower neighbor is darker (dark bg) / lighter (light bg);
non-edge cells = tone glyph of `tone * edgeBlend`. `render(frame, partial, out?)` wires:
resolve → validate frame (`data.length === w*h*4`) → rows/grid → reuse `out` if dims match
else allocate → sample → tone → `flip` → mode dispatch → `finalizeColors` → transparency
flags (mean alpha < 0.5 && background transparent → glyph space, flag bit0).

**Tests:** 256×64 horizontal gradient @ 40 cols tone/none → first col ` `, last col `@`,
monotonic non-decreasing ramp index across the row; invert flips indices exactly; light
background equals dark+invert for glyphs; vertical step → column of `|`; horizontal step →
row of `-` (or `_`); diagonal → `/` and `\`; colorMode `fg` red frame → fg (255,0,0) ±1;
depth 16 maps red to palette index 9 or 1 (document which by linear distance) and 256 → 196;
`out` reuse returns the same object; frame length mismatch throws RangeError.

**Done:** green; commit `feat(engine): color, tone and edge modes, render()`.

---

## Wave 4 — Remaining modes + serializers (parallel)

### T1.5 · Braille and block modes

**Owns:** `packages/engine/src/modes/{braille,blocks,subcells}.ts` + tests; wires into
`render.ts` dispatch (the only shared edit; one `case` per mode). **Reads:** T1.1–T1.4.
**Spec:** §3.2 rows *Mode: braille*, *Mode: blocks*.

**Contract:** `subcellBits(field, tone, cell, grid, opts, flip, relative): number` shared
helper — thresholds each subcell (absolute 0.5 after `quantize` with the chosen dither for
`none`/`fg`; relative to the cell mean with no dither for `fg+bg`), XORs `flip`. Braille: dot
bit order 1,2,3,7 left column (bits 0,1,2,6), 4,5,6,8 right (bits 3,4,5,7); `0x2800 + mask`;
empty cell is `0x2800`. Blocks: half (`" ▀▄█"` by 2-bit mask), quadrant table for all 16
masks (U+2580–259F), sextant `U+1FB00 + (mask-1) - corrections` with the four exceptions
(empty → space, left half ▌, right half ▐, full █). fg = mean of on-subcells, bg = mean of
off-subcells (fall back to cell mean when a side is empty).

**Tests:** all-white cell → `⣿`, all-black → `⠀`, single top-left dot → `⠁`; quadrant table
spot checks (`▘ ▝ ▖ ▗ ▚ ▞`), sextant endpoints `U+1FB00` (mask 1) and `U+1FB3B` (mask 62)
plus the 4 exceptions; fg/bg means on a half-red/half-blue cell in `fg+bg`; flip inverts
masks; `render` with each blockSet produces the right `cols × rows` and codepoint ranges.

**Done:** green; commit `feat(engine): braille and block modes`.

### T1.6 · Glyph atlas generator and glyph-match mode

**Owns:** `packages/engine/scripts/gen-atlas.ts`, `packages/engine/scripts/fonts/
DejaVuSansMono.ttf` + `LICENSE`, `packages/engine/data/{glyph-atlas,coverage}.json`,
`packages/engine/src/{atlas.ts,modes/glyph.ts}` + tests; wires `case 'glyph'` into
`render.ts`. **Reads:** T1.1–T1.4. **Spec:** §3.2 row *Mode: glyph*, *Ramp* (coverage).

**Contract:** atlas JSON `{ font: "DejaVu Sans Mono", cellW: 4, cellH: 8, threshold: 0.3,
glyphs: { [codepoint: string]: number /* 32-bit mask, bit = y*4+x */ } }`; coverage JSON
`{ font, glyphs: { [codepoint: string]: number /* 0..1 */ } }`. Generator: download
DejaVu Sans Mono from the `dejavu-fonts` GitHub release, commit TTF + license; register with
`@napi-rs/canvas` `GlobalFonts.registerFromPath`; render each codepoint of
`DEFAULT_OPTIONS.glyphSet` ∪ box drawing ∪ blocks at 64×128 px; area-average into 4×8;
`on` if ≥ 0.3; coverage = ink fraction. Commit the generated JSON (consumers never need
canvas). `loadAtlas(json, glyphSet): Atlas` precomputes masks + inverses, dedupes identical
masks to the lowest codepoint. `renderGlyph(ctx, atlas)`: per cell 4×8 target mask
(threshold at cell mean, `flip` applies `s → 1-s`), stage 1 Hamming shortlist of size
`max(1, round(workFactor * candidates))` using bucket counting (distances 0..32, no sort),
stage 2 color error per §3.2 (free fg/bg in `fg+bg`; fixed paper in `fg`/`none`); ties →
lowest codepoint. The default `render()` uses the committed atlas.

**Tests (synthetic atlas, no font):** atlas with space/full/left-half/right-half → left-white
cell picks `▌` with fg white bg black in `fg+bg`; uniform dark cell → space, uniform bright →
`█`; `workFactor 0` still returns a glyph; Hamming shortlist size formula; determinism across
two runs. **Real atlas:** JSON validates against the schema; contains `32`, `9608`, `9612`;
coverage of space is 0 and of `█` ≥ 0.95.

**Done:** green; `pnpm --filter @asciilab/engine gen:atlas` is idempotent (re-running
produces an identical file); commit `feat(engine): glyph atlas and glyph-match mode`.

### T1.7 · Serializers

**Owns:** `packages/engine/src/serialize/{text,ansi,html,svg}.ts` + tests.
**Reads:** T1.1, T1.4 (palettes). **Spec:** §3.3 (as amended).

**Contract:** exactly the four signatures in amended §3.3. ANSI 16-color index → SGR 30–37 /
90–97; 256 → `38;5;n`; truecolor → `38;2;r;g;b`; emit only on change; reset at row end only if
anything was emitted. HTML runs merge equal fg/bg; `inline:false` emits `class="c{n}"` and a
`<style>` block. SVG: `<rect>` per bg run, `<text x y textLength lengthAdjust="spacingAndGlyphs"
xml:space="preserve">` per fg run, page rect unless transparent.

**Tests:** `toText` trims trailing spaces, `keepTrailing` keeps them, row count = rows; two
red cells → one SGR; red then blue → two; 16-color red → `\x1b[91m` (or 31 — match T1.4's
nearest), 256 → `38;5;196`; `colorMode:'none'` → no `\x1b[` at all; HTML escapes `<&>`;
`toHtml(..., {colorMode:'none'})` has no `<span`; SVG parses as XML (regex check for
balanced root) and has `rows` `<text>` groups; property: strip-SGR(`toAnsi`) ===
`toText(keepTrailing)`.

**Done:** green; commit `feat(engine): text, ANSI, HTML, SVG serializers`.

---

## Wave 5 — Engine release candidate

### T1.9 · Public API, goldens, properties, coverage, build

**Owns:** `packages/engine/src/index.ts`, `packages/engine/test/{goldens/**,fixtures.ts,
golden.test.ts,properties.test.ts,api.test.ts}`, `packages/engine/vitest.config.ts`,
`packages/engine/tsdown.config.ts`, `packages/engine/README.md`. **Reads:** everything in
`packages/engine`. **Spec:** §3.5, §7.1, §2.1.

**Deliverables**
- `index.ts` exports exactly §3.5 (+ types). `api.test.ts` asserts `Object.keys(await
  import("../src/index.ts")).sort()` equals the expected list.
- `fixtures.ts`: gradient 256×64, checkerboard 64×64 (8px), filled circle 128×128, diagonals
  128×128, 4-color quadrants 64×64, alpha gradient 128×32 — all generated in code.
- `golden.test.ts`: matrix `fixture × mode(5) × dither{none, bayer4, floyd-steinberg} ×
  color{none, fg}` at 48 cols → `toText` (and `toAnsi` for `fg`) compared to
  `test/goldens/<fixture>-<mode>-<dither>-<color>.txt`; `UPDATE_GOLDENS=1` writes. First run
  writes; the task eyeballs at least one golden per mode (paste 3 into the report) before
  committing them.
- `properties.test.ts` (fast-check, 200 runs each): dims = `rowsFor`; invert involution in
  tone mode; monotonic ramp index vs luma (dither none); `toText` rows; ANSI strip equals text;
  `parseFlf(serialize(parseFlf(standard)))` header round-trip (add a tiny `serializeHeader`
  test helper — not exported).
- `vitest.config.ts`: coverage v8, `include: ["src/**"]`, `exclude: ["src/figlet/
  fonts.generated.ts"]`, `thresholds.lines: 90`.
- `tsdown.config.ts` builds `dist/index.js` + `dist/index.d.ts`; `pnpm --filter
  @asciilab/engine build` then `node -e "import('./packages/engine/dist/index.js').then(m =>
  console.log(Object.keys(m).sort().join()))"` prints the §3.5 list; `npm pack --dry-run` lists
  only `dist/**`, `README.md`, `LICENSE`, `package.json`.
- `packages/engine/README.md`: install, 20-line usage, API table, link to spec.

**Done:** coverage ≥ 90% lines (paste the summary); build + pack checks pass; commit
`feat(engine): public API, goldens, property tests, build`.

---

## Wave 6 — Web plumbing + CLI (parallel)

### T2.2 · Worker protocol, engine worker, client

**Owns:** `apps/web/src/worker/{protocol,handler,engine.worker,client}.ts` + tests.
**Reads:** `packages/engine/src/index.ts`, T2.1 `vite.config.ts`. **Spec:** §4.5, §4.8
(worker crash), §3.6.

**Contract:** `protocol.ts` discriminated unions `ToWorker = {type:'frame', id, frame,
options} | {type:'options', id, options}`, `FromWorker = {type:'cells', id, cells, ms} |
{type:'error', id, message}`, type guards, `transferablesOf(cells|frame)` (deduped buffers).
`handler.ts`: `createHandler()` returns `(msg) => FromWorker` holding the last frame; options
before any frame → error. `engine.worker.ts`: wires `self.onmessage` → handler →
`postMessage` with transferables (cast `self` to a minimal typed interface; no WebWorker lib).
`client.ts`: `EngineClient` with injectable `WorkerLike` factory; `render(frame|null,
options): Promise<CellBuffer | null>`; keeps its own copy of the frame; latest-wins (a
superseded request resolves `null`); on `onerror`: terminate, recreate, resend frame + current
request once, second crash rejects `WorkerCrashError`; engine `RangeError` → rejects
`RenderError(message)`.

**Tests (Vitest, node env, real engine):** handler renders a 16×16 frame to the right dims;
options-only after frame re-renders; options-before-frame errors; invalid options → error
message. Client with a fake worker: first render sends `frame` with a *copy*; two overlapping
renders → first resolves null; crash once → recovers; crash twice → rejects; `RangeError`
from worker → `RenderError`.

**Done:** green; commit `feat(web): worker protocol, engine worker, latest-wins client`.

### T2.3 · Sources, store, URL state

**Owns:** `apps/web/src/sources/{FrameSource,ImageSource}.ts`, `apps/web/src/state/{store,
urlState,sections}.ts`, `apps/web/src/lib/measureFont.ts` + tests. **Reads:** engine
`index.ts`, T2.1 `strings.ts` (append your strings). **Spec:** §4.3 (controls list), §4.5,
§4.8 (decode/CORS/downscale/too-large copy), §3.6, §9.

**Contract:**
- `FrameSource { readonly info: SourceInfo; current(): Frame; dispose(): void }`;
  `ImageSource.fromBlob(blob, name)`, `ImageSource.fromUrl(url)`; `checkSize(w,h): {kind:'ok'}
  | {kind:'downscale', w, h} | {kind:'too-large'}` (long edge ≤ 4096, ≤ 64 MP); decode via
  `createImageBitmap` (blob) or `<img crossorigin="anonymous" referrerpolicy="no-referrer">`
  (URL); `SourceError` with `kind: 'decode' | 'url' | 'too-large'` and the exact §4.8 copy;
  thumbnail = `OffscreenCanvas.convertToBlob` at ≤ 160 px.
- `store.ts` (Zustand): `settings: Settings` (`options: RenderOptions`, `aspectAuto`,
  `rampPreset: keyof typeof RAMPS | 'custom'`, `calibrate`, `font: 'jetbrains' | 'system' |
  'courier'`, `fontSize`, `banner: { text, font, layout, width, align }`), runtime (`tab`,
  `source: SourceInfo | null`, `cells`, `bannerCells`, `status`, `renderMs`, `error`,
  `notice`, `flash`, `theme`, `plainText`, `measuredAspect`, `calibratedRamp`, `helpOpen`),
  actions `setOptions`, `setSettings`, `resetSection(name)`, `resetAll`. `sections.ts` maps the
  seven §4.3 sections to option keys. `effectiveOptions(state): RenderOptions` applies
  measured aspect, calibrated ramp, and coerces `fg+bg` → `fg` for tone/edge.
- `urlState.ts` (zod/mini): short keys `m c a r rc cal i b k g d ds col dep bg bs et eb wf f
  fs`; per-key safe parse with default fallback; `parseSettings(search): Settings`,
  `serializeSettings(settings): string` omitting defaults; `useUrlSync()` writes
  `history.replaceState` on `/` only (debounced 100 ms). Never includes image bytes.
- `measureFont.ts`: `measureCell(ctx, family, px): { w, h, aspect }` using 100× "M" and
  `lineHeight 1.2`; `measureCoverage(ctx, glyphs, family): Record<number, number>`;
  injectable `ctx` for tests.

**Tests:** `checkSize` three branches; `nameFromUrl`; `effectiveOptions` coercion;
`resetSection('tone')` touches only tone keys; URL round-trip for non-default settings, empty
query for defaults, garbage keys ignored, `col=fg%2Bbg` decodes; `measureCell` with a fake
ctx returns `aspect` in 0.3..0.8.

**Done:** green; commit `feat(web): image sources, store, URL state, font measurement`.

### T3.1 · CLI

**Owns:** `packages/cli/**`, `.github/workflows/ci.yml` (append two steps only).
**Reads:** engine `dist/index.d.ts` (build it first), spec §5, §7.3, §9.

**Contract:** `src/args.ts` (`util.parseArgs`, strict, `allowPositionals`) → `Command =
{kind:'help'} | {kind:'version'} | {kind:'fonts'} | {kind:'image', input, out?, options:
Partial<RenderOptions>, color?, depth?, debug} | {kind:'text', text, font, layout, width?,
align, out?, debug}`; `UsageError` → exit 2 with the error, a usage excerpt, and "Run
asciilab --help". Every option in §5. Negative numbers (`--brightness -0.2`) parse. `src/io.ts`:
`detectDepth(env)`, `defaultColumns(stream)`, `defaultColor(stream)`, `readInput(path|'-')`,
`decode(buffer, label): Frame` via `sharp().ensureAlpha().raw()`, `writeOutput(cells, opts,
format, path)` (txt/ansi/html/svg via engine; png via `toSvg` → `sharp(Buffer).png()`,
DejaVu Sans Mono, cellW 9.6/cellH 19.2 at 16 px, documented font requirement). Defaults
when `--out` is a file: color `fg`, depth `truecolor`. `src/cli.ts`: shebang, dispatch, exit
codes 0/1/2, `--debug` stacks, EPIPE-safe stdout, version from `package.json`. `fonts` lists
`Object.keys(FONTS)`; `text` → `renderFiglet` + `figletToCells`. `tsdown.config.ts`
`platform: node`, external `sharp` and `@asciilab/engine`.

**Tests:** unit for `args` (table of valid/invalid), `io` (depth/columns/color/decode);
`test/global-setup.ts` builds engine (if `dist` missing) and cli; spawn tests: each mode's
stdout === `toText(render(frame, opts)) + "\n"` on a generated 64×64 gradient PNG (piped,
`COLORTERM=''`, `TERM=xterm`); `-c fg` emits `\x1b[`; `-` stdin; `--out` per extension
with magic bytes (`89 50 4E 47`, `<svg`, `<pre`, `\x1b[`), stdout empty; `--help` 0; bad flag
2 + "Unknown option"; missing file 1 + path in stderr; `fonts` matches `FONTS` keys;
`text "Hi"` matches `renderFiglet`. `scripts/smoke.sh`: pack engine + cli, `npm i` both
tarballs in a temp dir, run `npx asciilab fixture.png`, assert 20 lines. CI: append
`pnpm --filter asciilab typecheck` and `bash packages/cli/scripts/smoke.sh` after build.

**Done:** green incl. smoke; commit `feat(cli): asciilab command-line interface`.

---

## Wave 7 — Web core UI

### T2.4 · Output pane, status, dropzone, render loop

**Owns:** `apps/web/src/components/{OutputPane,StatusLine,Dropzone,SourcePanel}.tsx`,
`apps/web/src/app/{controller.ts,useRenderLoop.ts}`, `apps/web/src/workspaces/
ImageWorkspace.tsx`, `apps/web/scripts/gen-fixtures.ts`, `apps/web/public/demo.png`,
`apps/web/e2e/fixtures/*.png`, `apps/web/e2e/render.spec.ts`. Edits `App.tsx` to mount
`ImageWorkspace` on `/`. **Reads:** T2.1–T2.3. **Spec:** §4.3 regions 1–2 (as amended:
demo autoload), §4.5, §4.7 (role, aria-live), §4.8.

**Deliverables**
- `gen-fixtures.ts` (pngjs): `public/demo.png` 800×533 procedural scene; `e2e/fixtures/
  sample.png` 64×64 (quadrants + gradient circle), `wide.png` 5000×100, `alpha.png` 64×64
  with alpha gradient, `corrupt.png` (random bytes). Committed outputs; script is idempotent.
- `controller.ts`: singleton `EngineClient`; `loadBlob/loadUrl/loadDemo/clear`; keeps the
  `Frame` outside the store; `useRenderLoop()` subscribes to `effectiveOptions` + frame
  version, debounces 16 ms, calls `client.render`, writes `cells/renderMs/status`, auto-halves
  `workFactor` when glyph mode takes > 500 ms (notice with new value), maps errors to §4.8.
- `OutputPane`: `<pre id="output" role="img" aria-label=...>` updated imperatively via ref
  (`innerHTML` from `toHtml` with the outer `<pre>` stripped, or `textContent` when plain
  text / colorMode none); font-size slider 8–24; "Show as plain text" toggle swaps to a
  selectable `<pre>` without `role="img"`; `data-render-seq` increments per render; background
  per `options.background` (checkerboard for transparent).
- `StatusLine`: `aria-live="polite"`, "`{cols} × {rows} cells · {mode} · {ms} ms`", plus
  `notice`/`flash`; `role="alert"` for errors.
- `Dropzone`/`SourcePanel`: drag (whole panel), click → hidden input, window paste (image
  items or URL text when focus is not in a field), URL field + Load; thumbnail, name,
  `w × h`, Replace, Clear; empty state with "Load demo image". Auto-load demo on first visit
  to `/`.

**E2E (`render.spec.ts`):** `/` shows status with cells within 2 s; Clear → empty state →
Load demo → renders; `setInputFiles(sample.png)` → `data-render-seq` increments and output
changes; `wide.png` → downscale notice text; `corrupt.png` → decode error text; URL load of a
route-aborted URL → CORS copy.

**Done:** green; commit `feat(web): output pane, source panel, render loop, fixtures`.

---

## Wave 8 — Controls + export (parallel)

### T2.5 · Controls

**Owns:** `apps/web/src/components/ui/{Slider,Select,Segmented,Checkbox,Section}.tsx`,
`apps/web/src/components/Controls/**`, `apps/web/e2e/controls.spec.ts`. Edits
`ImageWorkspace.tsx` to mount `<Controls/>` in region 3. **Reads:** T2.3 store/sections.
**Spec:** §4.3 region 3 (order and contents), §3.2 *Color* (fg+bg disabled for tone/edge),
§9.

**Deliverables:** seven sections in spec order, each with a Reset; global Reset; Mode as a
labelled radio group; Size: columns slider 8–400, aspect Auto checkbox + manual slider
0.3–0.8, preview font Select; Characters: ramp preset Select (RAMPS keys + Custom), custom
ramp text input (≥ 2 unique chars else inline validation and keep previous), Calibrate
toggle; Tone: Invert, Brightness/Contrast −1..1, Gamma 0.2–5, Background segmented;
Dither: kind Select + Strength slider (hidden for `none`); Color: mode segmented (fg+bg
disabled with reason for tone/edge), Depth Select; Mode-specific: edge threshold/blend,
block set, work factor — shown only for the active mode. All inputs labelled; Radix sliders
have `aria-label`.

**E2E:** switching each of 5 modes changes output; columns slider changes status cols; invert
changes output; custom ramp "ab" accepted, "a" rejected with message; fg+bg disabled on tone;
URL updates after a change (`?m=braille` present) and reload restores the control states;
section Reset reverts only its controls.

**Done:** green; commit `feat(web): controls panel`.

### T2.6 · Export bar

**Owns:** `apps/web/src/export/{clipboard,download,rasterize,html}.ts`,
`apps/web/src/components/{ExportBar,DownloadMenu,CopyFallbackDialog}.tsx`,
`apps/web/e2e/export.spec.ts` + unit tests. Edits `ImageWorkspace.tsx` to mount
`<ExportBar/>` under the output. **Reads:** T2.3, T2.4. **Spec:** §4.3 export bar, §4.5
share link, §4.8 clipboard denied, §3.3.

**Deliverables:** `copyText/copyAnsi/copyHtml` via Clipboard API, fallback = dialog with a
preselected readonly textarea and the §4.8 copy; `download(kind)` for txt/ansi/html/svg
(HTML export is a full document with `<style>`), filenames `<base>-asciilab.<ext>`;
`rasterize.ts`: `planRaster(cols, rows, metrics, scale)` (pure, unit-tested) +
`rasterizePng(cells, opts)` on `OffscreenCanvas` with scale 1/2/4, transparent option, cell
bg rects then glyph runs, clamps to 16 384 px / 16.7 MP with the §4.8-style notice;
`DownloadMenu` = button with `aria-expanded` + panel (Esc/click-outside close, focus first
item), PNG scale radios + Transparent checkbox; Share link copies `location.href` with a
flash.

**Unit tests:** `planRaster` dims; filename; HTML doc wrapper; clipboard fallback path with a
rejected `writeText`. **E2E (chromium for clipboard):** Copy text → clipboard equals
`toText(render(decodedFixture, parseSettings(url)))` (decode fixture with pngjs in-test, set
`?a=0.5&c=48` so aspect is deterministic); each download's magic bytes/root; PNG 2× width =
2 × 1× width; transparent PNG has color type 6; Share link puts a URL in the clipboard that
parses to the same settings.

**Done:** green; commit `feat(web): copy, download, PNG rasterizer, share link`.

---

## Wave 9 — Text tab, shortcuts, a11y

### T2.7 · Text banner tab

**Owns:** `apps/web/src/workspaces/TextWorkspace.tsx`, `apps/web/src/components/
BannerControls.tsx`, `apps/web/e2e/text.spec.ts`. Edits `App.tsx` route `/text`.
**Reads:** T2.4–T2.6, engine figlet API. **Spec:** §4.3 text tab, §3.4.

**Deliverables:** textarea (default "asciilab"), font Select with a live two-letter preview
per option (parsed fonts cached in a `Map`), layout segmented (full/fitted/smushed), width
slider 20–400, align segmented → `renderFiglet` on the main thread → `figletToCells` →
`OutputPane` + `ExportBar` (share link hidden on `/text`; colorMode none).

**E2E:** typing changes output; each font changes output; width 20 wraps; downloads work.

**Done:** green; commit `feat(web): FIGlet text banner tab`.

### T2.8 · Keyboard shortcuts, help dialog, accessibility, error polish

**Owns:** `apps/web/src/lib/shortcuts.ts`, `apps/web/src/components/ShortcutsDialog.tsx`,
`apps/web/e2e/{keyboard,a11y,errors}.spec.ts` + unit tests. Edits `App.tsx` (keydown),
`Header.tsx` (`?` button). **Reads:** all web components. **Spec:** §4.7, §4.8.

**Deliverables:** `resolveShortcut(e): Action | null` (pure; ignores text fields and
Ctrl/Cmd combos); actions per §4.7; `ShortcutsDialog` (Radix Dialog) listing them; a11y
sweep: every control labelled, focus order sensible, `role="img"` + plain-text toggle, live
region, color contrast per tokens, `prefers-reduced-motion`. Every §4.8 row has a visible
path and copy.

**Tests:** unit table for `resolveShortcut`. **E2E:** Tab reaches every control (chromium);
`2` → edge, `]` → +10 cols, `I` → invert, `?` opens dialog, Esc closes; keys inside the URL
field do nothing; axe on `/`, `/text`, `/privacy` zero serious/critical in dark and light;
errors spec covers decode/CORS/downscale copy end-to-end.

**Done:** green; commit `feat(web): keyboard shortcuts, help, accessibility pass`.

---

## Wave 10 — Launch surface (parallel)

### T2.9 · Landing content, SEO assets, privacy test, visual snapshots, deploy

**Owns:** `apps/web/src/pages/{Landing,Faq}.tsx`, `apps/web/scripts/{gen-og,gen-icons}.ts`,
`apps/web/public/{og.png,favicon.svg,favicon-32.png,apple-touch-icon.png,robots.txt,
sitemap.xml}`, `apps/web/e2e/{privacy,visual}.spec.ts`, `apps/web/lighthouserc.json`,
`.github/workflows/deploy.yml`, `.github/workflows/ci.yml` (append Playwright steps).
Edits `App.tsx` (landing sections under the converter on `/`). **Spec:** §4.6, §4.9, §4.10,
§7.2, §7.4, §8.2–3, §8.6–7.

**Deliverables:** H1 "Image to ASCII art, in your browser" + value prop above the converter
on `/` (and "Text to ASCII banner" on `/text`); below: three feature rows with real ASCII
samples, how-it-works/privacy, FAQ (10 Q&A, `<details>`), footer. `gen-icons.ts`: hand-drawn
`favicon.svg` → PNGs via Playwright screenshot; `gen-og.ts`: load `/`, export PNG via the
app's rasterizer, composite onto 1200×630 with title. `robots.txt`, `sitemap.xml` from
`VITE_SITE_URL`. **Privacy spec:** after `networkidle`, record every `http(s)` request while
dropping a fixture, changing every control, and running every export; assert the set is
empty (blob:/data: ignored; build has `VITE_ANALYTICS=0`). **Visual spec:** 3 snapshots per
§7.2 (as amended), status masked, skipped off-Linux. `lighthouserc.json` thresholds
performance ≥ 0.9, a11y/best-practices/seo = 1 against `vite preview`, mobile preset;
documented command. `deploy.yml`: build with `VITE_SITE_URL`/`VITE_CF_BEACON_TOKEN` from repo
variables, `cloudflare/wrangler-action@v3` `pages deploy apps/web/dist --project-name
asciilab --branch ${{ github.head_ref || 'main' }}`, preview URL to the job summary; secrets
`CLOUDFLARE_API_TOKEN`, `CLOUDFLARE_ACCOUNT_ID`; one-time `wrangler pages project create
asciilab --production-branch main` documented in README. CI: `playwright install --with-deps
chromium webkit`, `test:e2e`, upload `playwright-report` on failure.

**Done:** all e2e green locally (visual skipped on macOS); Lighthouse thresholds met locally;
commit `feat(web): landing, SEO assets, privacy test, visual snapshots, deploy`.

### T3.2 · Release automation and repo hygiene

**Owns:** `.github/workflows/release.yml`, `.changeset/initial-release.md`,
`.github/ISSUE_TEMPLATE/{bug,feature}.yml`, `.github/PULL_REQUEST_TEMPLATE.md`,
`CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, `SECURITY.md`, `README.md` (full rewrite),
`docs/assets/screenshot.png`, `docs/LAUNCH.md`, `packages/*/package.json` metadata fields
only (`repository`, `homepage`, `bugs`). **Spec:** §6, §7.4 release, §8.

**Deliverables:** `release.yml` = Changesets 3 multi-job flow (select-mode → version PR /
pack → publish with `id-token: write` and npm trusted publishing, Node 24 so npm ≥ 11.5.1 →
umbrella `vX.Y.Z` GitHub release with notes from both CHANGELOGs, idempotent). Initial
changeset bumps both packages major → 1.0.0. README: hero screenshot (command in
`docs/LAUNCH.md`), 30-second usage for web/CLI/library, feature matrix, comparison table vs
chafa and Asciinova (claims sourced from spec §1.1, footnoted with date), privacy statement,
packages table, license. Contributor Covenant 2.1 full text with `[INSERT CONTACT]`
placeholder flagged for the maintainer. SECURITY.md: GitHub private vulnerability reporting.
`docs/LAUNCH.md`: one-time manual steps (npm org `asciilab`, bootstrap publish 0.0.0 +
deprecate, trusted publisher config per package, Cloudflare project + secrets, custom domain)
and the §8 Definition-of-Done checklist with the exact command/URL for each of the 10 items,
including the 10-image manual QA table.

**Done:** `release.yml` passes `action-validator` or `actionlint` (via `npx`/docker); `pnpm
changeset status` shows the pending major; commit `chore: release automation, README, repo
hygiene, launch checklist`.

---

## Wave 11 — Ship

### T4 · Definition-of-Done walk and fixes

**Owns:** anything needed to make §8 true; must list every file touched.
**Reads:** `docs/LAUNCH.md`, spec §8.

Run the checklist top to bottom: CI green on `main`; coverage ≥ 90%; deploy succeeds (requires
the maintainer to have created the Cloudflare project and secrets — if not present, stop and
report exactly what is needed); Lighthouse on the deployed URL; `npx asciilab photo.jpg` on
Node 22 in Docker; library import in a scratch Vite app; README sections; CSP header via
`curl -sI`; OG unfurl check; axe + shortcuts; release tagged; 10-image QA via a Playwright
script that screenshots each image in each mode into `docs/qa/` for human review. Fix
defects found; re-run. Report a table: item · command · result.

**Done:** every §8 item ✅ with evidence; commit `chore: v1 launch QA`.

---

## Wave map

```
W0  T0
W1  T1.1
W2  T1.2  T1.3  T1.8  T2.1
W3  T1.4
W4  T1.5  T1.6  T1.7
W5  T1.9
W6  T2.2  T2.3  T3.1
W7  T2.4
W8  T2.5  T2.6
W9  T2.7 → T2.8
W10 T2.9  T3.2
W11 T4
```

## Spec amendments made while planning (already applied to the spec)

1. Serializers take `colorMode` (CellBuffer does not carry it); exact signatures fixed.
2. `figletToCells(lines: string[])`, `renderFiglet` returns `string[]` with defaults.
3. `calibrateRamp(glyphs: string, coverage: Record<number, number>)`.
4. `invert` applied after quantization (exact involution).
5. CSP `connect-src` is `https://cloudflareinsights.com` (beacon POST target).
6. Pinned Vite 7.x / TS 5.9.x / pnpm 10.x; `npm i -g pnpm@10` (Node 26 has no corepack).
7. Demo image and fixtures are procedurally generated (no licensing).
8. Visual snapshot baselines are Linux-generated in CI.
