# asciilab — Design Spec

**Date:** 2026-10-28
**Status:** Draft for review
**Owner:** vidit19sharma
**License:** MIT (all packages)

## 1. Product

asciilab is a free, open-source, privacy-first converter that turns images and text into
high-quality text art (ASCII, ANSI-color, braille, Unicode blocks, FIGlet banners). Everything
runs on the user's device. Nothing is uploaded.

Surfaces, in priority order:

1. **Web app** (`apps/web`) — the product. Static site, zero backend.
2. **CLI** (`packages/cli`, npm `asciilab`) — same engine, for terminals and scripts.
3. **Library** (`packages/engine`, npm `@asciilab/engine`) — same engine, for developers.

### 1.1 Positioning (from competitor research)

No existing tool combines: in-browser privacy, all render modes (tone / edge / braille /
blocks / glyph-match), proper dithering, color, and exports that preserve the chosen mode.
Web competitors are either stale (text-image.com), basic (manytools), or pivoted to paid AI
image generation (Asciinova, asciiart.com). The CLI quality bar is `chafa`; the web has no
equivalent. asciilab is the web equivalent of chafa, plus a text-banner tool, plus exports.

### 1.2 v1 scope

**In:**

- Inputs: PNG, JPEG, WebP, GIF (first frame), BMP, AVIF (where the browser decodes it) via
  drag-drop, file picker, clipboard paste, or URL (same-origin or CORS-enabled).
- Text banners via FIGlet `.flf` fonts (bundled set).
- Render modes: Tone, Edge, Braille, Blocks (half / quadrant / sextant), Glyph-match.
- Controls: columns, cell aspect (auto-measured from preview font, overridable), charset preset
  + custom ramp, invert, brightness, contrast, gamma, dithering, color mode, color depth,
  background handling (dark / light / transparent), preview font family.
- Exports: copy as plain text / ANSI / HTML; download `.txt`, `.ansi`, `.html`, `.svg`, `.png`
  (1×, 2×, 4×; opaque or transparent background).
- Shareable URL that encodes settings only (never image bytes).
- CLI with parity for all modes and text/ANSI/HTML/SVG/PNG output.
- Landing content, privacy page, keyboard shortcuts, WCAG 2.2 AA.

**Out (explicitly deferred, with the seams for them designed in — see §3.6):**

- Video files, webcam, animated GIF/APNG/WebM export.
- Manual glyph painting / editing of the output.
- PWA / offline install.
- Accounts, payments, hosted API, AI generation.

## 2. Architecture decision

Chosen: **pure-TypeScript engine shared across web, CLI, and library** (approach A).

Rejected: Rust → WASM core (approach B) — 2–3× slower to ship, two toolchains, no benefit for
still images; the only hot path (glyph-match) is isolated behind one function and can be ported
later without API change. Rejected: wrapping `chafa-wasm` (approach C) — LGPL, emits ANSI
strings rather than a cell buffer, kills clean HTML/SVG/PNG export and future video.

### 2.1 Non-negotiable engine rules

1. **Cell buffer, not strings.** Internal representation is `CellBuffer` (§3.1). Strings exist
   only in serializers.
2. **DOM-free engine.** `@asciilab/engine` imports nothing from `window`, `document`, or Node.
   Input is RGBA typed arrays; output is `CellBuffer`. Font measurement and rasterization live
   in the web app and CLI, not the engine.
3. **Stateless, frame-oriented API.** `render(frame, options) → CellBuffer`. A still image is a
   one-frame sequence. No module-level mutable state.
4. **Deterministic.** Same input + options → byte-identical output. No `Math.random` without a
   seeded PRNG in options. Ties broken by lowest codepoint.
5. **Allocation-aware.** Hot paths accept optional pre-allocated output buffers and never
   allocate per cell.
6. **Zero runtime dependencies** in `@asciilab/engine`.

## 3. Engine (`packages/engine`)

### 3.1 Core types

```ts
export interface Frame {
  width: number;          // px
  height: number;         // px
  data: Uint8ClampedArray; // RGBA, length = width*height*4, sRGB, straight alpha
}

export interface CellBuffer {
  cols: number;
  rows: number;
  glyphs: Uint32Array;    // Unicode codepoint per cell, length = cols*rows
  fg: Uint8Array;         // RGB per cell, length = cols*rows*3 (ignored when colorMode === 'none')
  bg: Uint8Array;         // RGB per cell, length = cols*rows*3 (ignored unless colorMode === 'fg+bg')
  flags: Uint8Array;      // bit0 = transparent cell (source alpha < 0.5 and background === 'transparent')
}

export type Mode = 'tone' | 'edge' | 'braille' | 'blocks' | 'glyph';
export type Dither = 'none' | 'bayer2' | 'bayer4' | 'bayer8' | 'noise' | 'floyd-steinberg' | 'atkinson';
export type ColorMode = 'none' | 'fg' | 'fg+bg';
export type ColorDepth = 16 | 256 | 'truecolor';
export type Background = 'dark' | 'light' | 'transparent';
export type BlockSet = 'half' | 'quadrant' | 'sextant';

export interface RenderOptions {
  mode: Mode;
  cols: number;                 // 8..400 (web hard cap 400; CLI no cap)
  cellAspect: number;           // cell width / cell height, typically 0.4..0.6; default 0.5
  ramp: string;                 // ordered darkest→lightest for 'dark' background; ≥2 chars
  invert: boolean;
  brightness: number;           // -1..1, default 0
  contrast: number;             // -1..1, default 0
  gamma: number;                // 0.2..5, default 1
  luma: 'rec709' | 'rec601';    // default rec709
  dither: Dither;
  ditherStrength: number;       // 0..1, default 1
  colorMode: ColorMode;
  colorDepth: ColorDepth;
  background: Background;
  blockSet: BlockSet;           // used when mode === 'blocks'
  edgeThreshold: number;        // 0..1, used when mode === 'edge'; default 0.25
  edgeBlend: number;            // 0..1, how much tone shows through edges; default 0.6
  glyphSet: string;             // candidate glyphs for mode === 'glyph'; default = ramp + blocks + box drawing
  workFactor: number;           // 0..1, glyph-match search budget; default 0.5
  seed: number;                 // for 'noise' dither; default 0
}

export const DEFAULT_OPTIONS: RenderOptions;
export function render(frame: Frame, options: Partial<RenderOptions>, out?: CellBuffer): CellBuffer;
```

Validation: `render` throws `RangeError` with a field name on out-of-range values. The web app
validates before calling, so the throw is a programming-error guard, not a UX path.

### 3.2 Pipeline stages (one module each, `src/`)

| Stage | Module | Behavior |
|---|---|---|
| Geometry | `geometry.ts` | `rows = round(cols * (frame.height / frame.width) * cellAspect)`, min 1. Subcell grid per mode: tone/edge 1×1, braille 2×4, blocks half 1×2 / quadrant 2×2 / sextant 2×3, glyph 4×8. |
| Resample | `sample.ts` | Area-average each subcell from source pixels in **linear light** (sRGB → linear LUT, 256 entries). Outputs per-subcell linear RGB + alpha. Alpha composited against `background` (dark = #000, light = #fff, transparent = kept, flag bit0 set when alpha < 0.5). |
| Tone | `tone.ts` | Luma (709: 0.2126/0.7152/0.0722; 601: 0.299/0.587/0.114) on linear RGB → apply contrast (around 0.5), brightness (offset), gamma (`pow`), invert, clamp 0..1. `background === 'light'` flips the ramp direction (lightest glyph = darkest luma) instead of inverting the image. |
| Dither | `dither.ts` | Operates on the tone value before quantization to ramp index. Bayer 2/4/8 threshold matrices; `noise` = seeded hash per cell (temporally stable); Floyd-Steinberg and Atkinson error diffusion across cells in scan order. `ditherStrength` scales the perturbation. |
| Ramp | `ramps.ts` | Presets: `standard` (`" .:-=+*#%@"`), `long70` (classic 70-char), `blocks` (`" ░▒▓█"`), `minimal` (`" .oO@"`), `binary` (`" #"`). `calibrateRamp(glyphs, coverage: Map<string, number>)` sorts glyphs by measured ink coverage; coverage is measured by the host (web: offscreen canvas; CLI: precomputed table for DejaVu Sans Mono shipped in `data/coverage.json`). Uncalibrated custom ramps are used in the order given. |
| Mode: tone | `modes/tone.ts` | `glyph = ramp[round(tone * (ramp.length - 1))]`. |
| Mode: edge | `modes/edge.ts` | Sobel on the per-cell tone field (3×3). Magnitude ≥ `edgeThreshold` → directional glyph by orientation bucket: `-` (0°), `/` (45°), `\|` (90°), `\` (135°), with `_` for strong horizontal at bottom-weighted cells. Below threshold → tone glyph for `tone * edgeBlend` (non-edge regions are dimmed so edges stand out; `edgeBlend = 0` gives pure line art). |
| Mode: braille | `modes/braille.ts` | 2×4 subcells each thresholded (after dither) to a dot; codepoint = `0x2800 + bitmask` using the standard dot order (1,2,3,7 left column; 4,5,6,8 right). Empty cell = `0x2800` (not space) so columns align. |
| Mode: blocks | `modes/blocks.ts` | Subcells thresholded to a bitmask; map to U+2580–259F for half/quadrant, U+1FB00–1FB3B (Symbols for Legacy Computing) for sextant. fg = mean of "on" subcells, bg = mean of "off" subcells. |
| Mode: glyph | `modes/glyph.ts` | chafa-style staged search. Per cell: the 4×8 subcell tones thresholded at the cell mean → 32-bit mask. Precomputed atlas (`data/glyph-atlas.json`, generated by `scripts/gen-atlas.ts` from DejaVu Sans Mono): each candidate glyph's 4×8 mask and its inverse. Only glyphs present in the atlas are candidates; characters in `glyphSet` that are missing from the atlas are ignored. Stage 1: popcount(XOR) Hamming shortlist of size `max(1, round(workFactor * atlas.length))`. Stage 2: for shortlisted candidates (including inverses), compute fg = mean of subcells under the glyph's ink and bg = mean under its paper, pick lowest squared color error. An inverse winner is stored as the same glyph with fg/bg swapped; no extra flag. |
| Color | `color.ts` | For subcell modes (braille, blocks, glyph): `fg` = mean linear RGB of ink subcells → sRGB; `bg` = mean of paper subcells. For 1×1 modes (tone, edge): `fg` = cell mean color; `fg+bg` is treated as `fg` and the UI disables the option. When `colorMode !== 'fg+bg'`, `bg` = background color. Quantize to 16 (standard VGA palette) or 256 (xterm cube + grays) by nearest in linear RGB; truecolor passes through. |
| Orchestrator | `render.ts` | Wires stages; allocates or reuses `out`. |

### 3.3 Serializers (`src/serialize/`)

| Output | Function | Notes |
|---|---|---|
| Plain text | `toText(cells): string` | Glyphs only, `\n` rows, trailing spaces trimmed per row unless `keepTrailing`. |
| ANSI | `toAnsi(cells, { depth }): string` | 16-color: SGR 30–37/90–97 (fg) and 40–47/100–107 (bg); 256: `38;5;n` / `48;5;n`; truecolor: `38;2;r;g;b` / `48;2;r;g;b`. SGR emitted only when fg/bg change from the previous cell; `\x1b[0m` at row end. |
| HTML | `toHtml(cells, { inline: boolean }): string` | `<pre class="asciilab">` with one `<span style="color:..;background:..">` per run of identical colors. Escapes `<`, `>`, `&`. `inline: false` emits classes + a `<style>` block instead of inline styles. |
| SVG | `toSvg(cells, { fontFamily, fontSize, cellW, cellH, background }): string` | One `<rect>` per bg run, one `<text>` per fg run with `xml:space="preserve"` and `textLength` for exact column alignment. Host supplies measured `cellW/cellH`. |

PNG is **not** in the engine (needs a canvas). Web: `apps/web/src/export/rasterize.ts` draws
the cell buffer to an `OffscreenCanvas` using measured metrics and `devicePixelRatio`. CLI:
`toSvg` → `sharp(svgBuffer).png()`.

### 3.4 FIGlet (`src/figlet/`)

- `parseFlf(text): FigFont` — header (hardblank, height, baseline, max length, old layout,
  comment lines, print direction, full layout, codetag count), character table for ASCII 32–126
  plus German umlauts, code-tagged extra characters. Rejects compressed `.flf` (zip) with a
  clear error; the bundler pre-extracts.
- `renderFiglet(text, font, { layout: 'full' | 'fitted' | 'smushed' | 'controlled', width?, align })`
  implementing the six horizontal smushing rules and vertical layout per `figfont.txt`. Lines
  longer than `width` wrap at word boundaries. Returns `string[]` (lines) and a `CellBuffer`
  adapter (`figletToCells`) so the same exports apply.
- Bundled fonts (public domain / FIGlet-distributed, verified before inclusion): `standard`,
  `slant`, `small`, `big`, `banner`, `block`, `shadow`, `script`, `mini`, `digital`. Stored as
  `.flf` text imported as strings; ~150 KB total.

### 3.5 Public API (`src/index.ts`)

Exports: `render`, `DEFAULT_OPTIONS`, all types, `RAMPS`, `calibrateRamp`, `toText`, `toAnsi`,
`toHtml`, `toSvg`, `parseFlf`, `renderFiglet`, `figletToCells`, `FONTS` (name → flf string),
`createCellBuffer(cols, rows)`. ESM only. `sideEffects: false`.

### 3.6 Seams for v1.1 (video / webcam) — designed now, not built

- `render` is per-frame and stateless → a video is `frames.map(f => render(f, o, reused))`.
- Serializers accept a single `CellBuffer`; an animated exporter will take `CellBuffer[]` + fps.
- `noise` and Bayer dithers are temporally stable; UI will default to them for video.
- Web `FrameSource` interface (`apps/web/src/sources/`) with one implementation (`ImageSource`)
  in v1; `VideoFileSource` and `WebcamSource` slot in later without touching the renderer.
- Worker protocol already carries a frame id and supports latest-wins cancellation.

## 4. Web app (`apps/web`)

### 4.1 Stack

Vite 7, React 19, TypeScript 5.9 (strict), Tailwind CSS v4, Zustand (state), Zod v4 `zod/mini`
(URL-state schema), Vite `?worker` module worker. No UI component library; custom components
with Radix primitives only where accessibility is hard (slider, select, tabs, dialog). Fonts:
a bundled monospace (JetBrains Mono, OFL) as the default preview font, plus system monospace
options.

### 4.2 Routes

Single-page app with hash-free client routing (`/`, `/text`, `/privacy`, `/about`) via a
10-line path switch — no router library. Cloudflare Pages `_redirects` sends all paths to
`index.html`. `/` and `/text` share the shell; the tab bar switches between them.

### 4.3 Layout

Three regions, responsive (stacked under 900 px):

1. **Source** (left, 280 px): dropzone (drag, click, paste, URL field), source thumbnail,
   file name + dimensions, "Replace" and "Clear".
2. **Output** (center, flexible): `<pre id="output">` whose `innerHTML`/`textContent` is set
   imperatively from `toHtml`/`toText` — React does not reconcile cells. Pan/zoom via CSS
   `font-size` slider (8–24 px) and native scroll. Live status line: "160 × 72 cells · tone ·
   14 ms". Empty state shows a demo image (bundled, 1 click to load).
3. **Controls** (right, 320 px): grouped sections in this order — Mode · Size (columns, cell
   aspect auto/manual) · Characters (ramp preset / custom / calibrate toggle) · Tone (invert,
   brightness, contrast, gamma, background) · Dither (kind, strength) · Color (mode, depth) ·
   Mode-specific (edge threshold/blend; block set; glyph work factor). Reset button per section
   and global.

Export bar under the output: `Copy text` · `Copy ANSI` · `Copy HTML` · `Download ▾`
(.txt / .ansi / .html / .svg / .png 1×·2×·4×, transparent checkbox) · `Share link`.

Text banner tab (`/text`): text input (multiline), font select with live glyph preview,
layout (full / fitted / smushed), width, alignment; same Output and Export bar.

### 4.4 Visual direction

Dark-first, terminal-native aesthetic: near-black surfaces, one accent, monospace display
type, hairline borders, no gradients, no card shadows, no stock SaaS hero. Light theme via
`prefers-color-scheme` with a manual toggle. The output pane is the hero. Implementation is
delegated to a visual-engineering agent with the `design-taste-frontend` skill; this spec
fixes the information architecture above, not the pixels.

### 4.5 Data flow

```
drop/paste/file → createImageBitmap(blob)        ─┐
URL → <img crossorigin="anonymous"> → onload     ─┴→ bound (long edge ≤ 4096 px, else downscale,
show notice) → Frame (RGBA via OffscreenCanvas.getImageData) → postMessage to worker
{ id, frame (Transferable), options } → worker: render → postMessage { id, cells (Transferable) }
→ main: if id === latest → toHtml/toText → output.innerHTML → status line (aria-live)
```

- Worker holds the last `Frame` so option changes re-render without re-transferring pixels
  (`{ id, options }` only). A new image replaces it.
- Debounce slider input at 16 ms; latest-wins: results with stale `id` are dropped.
- Cell aspect auto-measure: render "M" × 100 in the chosen font to an offscreen canvas, read
  `measureText` width and line height; recompute on font change.
- URL state: `?m=tone&c=120&r=standard&inv=0&b=0&k=0&g=1&d=none&col=fg&dep=tc&bg=dark…`
  parsed with the Zod schema; unknown/invalid keys fall back to defaults silently. Never
  includes image data. `Share link` copies the current URL.

### 4.6 Privacy guarantee (verifiable)

- No backend. CSP (via `_headers`): `default-src 'self'; img-src 'self' data: blob: https:;
  connect-src 'self' https://static.cloudflareinsights.com; worker-src 'self'; script-src
  'self' https://static.cloudflareinsights.com; style-src 'self' 'unsafe-inline'`.
  `img-src https:` exists only so the URL input can load a user-supplied image via
  `<img crossorigin="anonymous">`; `connect-src` stays strict so no code path can POST bytes.
- Playwright test (analytics disabled) intercepts all requests after initial load, drops an
  image, changes every control, exercises every export, and asserts **zero** network requests
  occurred.
- Visible badge in the header: "Runs on your device. Nothing is uploaded." linking to
  `/privacy`, which explains the above in plain language, including the one caveat: loading
  by URL fetches the image from that URL, so that site sees a normal image request.

### 4.7 Accessibility and keyboard

- All controls are native or Radix; visible focus rings; labels tied to inputs.
- `#output` has `role="img"` with `aria-label="Text art output, {cols} by {rows} characters"`
  and a "Show as plain text" toggle that swaps colored spans for a selectable `<pre>`.
- `aria-live="polite"` status line announces render completion and errors.
- Shortcuts (documented in a `?` dialog): `O` open file, `V` paste, `C` copy text, `Shift+C`
  copy ANSI, `S` download menu, `1–5` switch mode, `[`/`]` columns ±10, `I` invert, `R` reset,
  `?` help. Disabled while focus is in a text field.
- WCAG 2.2 AA contrast for UI chrome in both themes; `prefers-reduced-motion` respected.

### 4.8 Errors (user-facing copy is specific, never "something went wrong")

| Condition | Behavior |
|---|---|
| Unsupported or corrupt file | Inline error in Source panel: "Couldn't decode `name`. Supported: PNG, JPEG, WebP, GIF, BMP, AVIF." |
| URL fetch fails / CORS | "That site doesn't allow direct loading. Download the image and drop it here instead." |
| Image > 4096 px long edge | Auto-downscale; notice: "Downscaled from 8000×6000 for speed. Output quality is unaffected at ≤400 columns." |
| Image > 64 MP or decode OOM | Refuse: "This image is too large to process in the browser (max 64 megapixels)." |
| Worker crash | Restart worker, retry once silently; on second failure show "Rendering failed. Reload the page; if it persists, open an issue (link)." |
| Glyph-match > 500 ms | Automatically halve `workFactor`, re-render, show notice with the new value. |
| Clipboard denied | Fall back to selecting the output text and prompting Ctrl/Cmd+C. |

### 4.9 Landing content and SEO

Above the fold on `/`: H1 "Image to ASCII art, in your browser", one-line value prop, the live
converter preloaded with the demo image. Below: three feature rows (modes, color + dithering,
exports), a "How it works / privacy" section, FAQ (10 Q&A targeting "image to ascii",
"photo to ascii art", "ascii art generator", "braille art", "ansi art", "figlet online"), and
footer with GitHub, npm, privacy, about. `<title>`, meta description, canonical, OG image
1200×630 (generated by the app itself), `sitemap.xml`, `robots.txt`, favicon set.

### 4.10 Analytics

Cloudflare Web Analytics beacon only (no cookies), loaded only in production builds. Custom
events: none in v1.

## 5. CLI (`packages/cli`)

```
asciilab <image> [options]            convert an image
asciilab text "Hello" [options]       render a FIGlet banner
asciilab fonts                        list bundled FIGlet fonts
asciilab --help | --version

Options (image):
  -m, --mode <tone|edge|braille|blocks|glyph>   default tone
  -w, --width <cols>                             default: terminal columns, or 80 when piped
  -a, --aspect <ratio>                           default 0.5
  -r, --ramp <preset|chars>                      default standard
  -i, --invert
      --brightness/--contrast/--gamma <n>
  -d, --dither <none|bayer2|bayer4|bayer8|noise|floyd-steinberg|atkinson>
  -c, --color <none|fg|fg+bg>                    default: fg when stdout is a TTY, none when piped
      --depth <16|256|truecolor>                 default: detected from $COLORTERM / $TERM
      --bg <dark|light|transparent>
      --block-set <half|quadrant|sextant>
      --edge-threshold/--edge-blend <n>
      --work-factor <n>
  -o, --out <file>                               format by extension: .txt .ansi .html .svg .png
      --debug                                    print stack traces
Options (text): -f, --font <name>  -l, --layout <full|fitted|smushed>  -w, --width  --align <left|center|right>
```

- Argument parsing with Node's built-in `util.parseArgs`; no CLI framework.
- Decode with `sharp` → raw RGBA → `Frame`. PNG output: `toSvg` → `sharp().png()` with
  DejaVu Sans Mono (documented as a system-font dependency for `.png`).
- Exit codes: 0 success, 1 runtime error (message on stderr), 2 usage error (+ help excerpt).
- Reads from stdin when `<image>` is `-`.
- Published as ESM with a `bin` entry; `npx asciilab photo.jpg` works with no install.

## 6. Repository

```
asciilab/
├── apps/web/                      Vite + React app
│   ├── public/                    favicon, og.png, robots.txt, sitemap.xml, _redirects, _headers (CSP)
│   ├── src/
│   │   ├── app/                   shell, routes, theme
│   │   ├── components/            Dropzone, OutputPane, Controls/*, ExportBar, StatusLine, ShortcutsDialog
│   │   ├── sources/               FrameSource, ImageSource
│   │   ├── worker/                engine.worker.ts, protocol.ts, client.ts
│   │   ├── export/                rasterize.ts (PNG), download.ts, clipboard.ts
│   │   ├── state/                 store.ts (zustand), urlState.ts (zod schema)
│   │   ├── strings.ts             all user-facing copy (i18n-ready)
│   │   └── pages/                 Landing sections, Privacy, About
│   └── e2e/                       Playwright specs + fixtures
├── packages/engine/
│   ├── src/                       per §3
│   ├── data/                      coverage.json, glyph-atlas.json (generated), fonts/*.flf
│   ├── scripts/                   gen-atlas.ts (@napi-rs/canvas, devDependency; writes coverage.json + glyph-atlas.json)
│   └── test/                      unit, properties, goldens/*.txt, fixtures/*.png
├── packages/cli/
│   ├── src/                       cli.ts, args.ts, io.ts
│   └── test/
├── docs/superpowers/specs/        this file
├── .changeset/
├── .github/
│   ├── workflows/ci.yml, release.yml, deploy.yml
│   └── ISSUE_TEMPLATE/bug.yml, feature.yml, PULL_REQUEST_TEMPLATE.md
├── biome.json  tsconfig.base.json  pnpm-workspace.yaml  package.json
├── README.md  LICENSE  CONTRIBUTING.md  CODE_OF_CONDUCT.md  SECURITY.md
```

Tooling: pnpm 10 workspaces; Biome 2 (lint + format); TypeScript strict with
`noUncheckedIndexedAccess`; `tsdown` builds the two publishable packages (ESM, `.d.ts`);
Changesets for versioning and changelogs; Node ≥ 22 engines field.

## 7. Testing

### 7.1 Engine (Vitest)

- **Unit tests** per module (geometry rounding, sRGB↔linear LUT, luma coefficients, each
  dither matrix, braille bit order, block codepoint tables, Hamming shortlist, ANSI SGR
  minimization, HTML escaping, FIGlet header parsing and each smush rule).
- **Golden tests**: synthetic fixtures generated in-test (no binary files): horizontal
  gradient 256×64, checkerboard, filled circle, diagonal edges, 4-color quadrants, alpha
  gradient. For each `(fixture, mode, dither, colorMode)` in a fixed matrix, `toText`/`toAnsi`
  output is compared to `test/goldens/*.txt`. Update via `pnpm test:update-goldens`; diffs are
  reviewed in PRs.
- **Property tests** (fast-check): output dims equal geometry for any frame size 1–512 px and
  cols 8–400; `invert` is an involution on glyph indices for tone mode; tone mapping is
  monotonic in input luma when dither is `none`; `toText(cells).split('\n').length === rows`;
  `toAnsi` stripped of SGR equals `toText`; `parseFlf(serialize(font))` round-trips the header.
- **FIGlet goldens**: for each bundled font, `renderFiglet("Hello, World! 123", font)` matches
  output captured once from reference `figlet` (committed as fixtures with the command used).
- Coverage target: 90 % lines on `packages/engine/src`, enforced in CI.

### 7.2 Web (Playwright, pinned Chromium + WebKit)

- Load `/`, demo image renders within 2 s, status line shows cell count.
- Drop fixture PNG → output changes; switch each of 5 modes → output changes each time.
- Copy text → clipboard equals `toText` of the current cells (computed in-test from engine).
- Each download format produces a file with the expected magic bytes / root element.
- URL round-trip: set options → copy share link → open in new page → same options.
- **Privacy**: route-intercept all requests; assert none after load except static assets.
- Keyboard: tab order reaches every control; shortcuts act; `?` opens help.
- Axe accessibility scan on `/`, `/text`, `/privacy` with zero serious/critical violations.
- 3 screenshot snapshots (empty state, rendered tone, rendered glyph with color) at 1280 px,
  dark theme, 0.5 % pixel tolerance.

### 7.3 CLI (Vitest, spawning the built binary)

Fixture PNG → each mode → stdout matches engine `toText` for the same options; `--out` for each
extension writes a valid file; `--help` exits 0; bad flag exits 2; missing file exits 1.

### 7.4 CI (GitHub Actions)

`ci.yml` on PR and `main`: install (frozen lockfile) → Biome check → `tsc -b` → Vitest with
coverage → build all → Playwright → upload artifacts. `deploy.yml`: Cloudflare Pages preview
on PR, production on `main`. `release.yml`: Changesets bot opens version PR; merge publishes
`@asciilab/engine` and `asciilab` to npm via OIDC trusted publishing and tags a GitHub release.

## 8. Definition of done (v1 launch)

1. All tests in §7 pass in CI on `main`; engine coverage ≥ 90 %.
2. Production deploy on Cloudflare Pages at a custom domain over HTTPS; deep links work.
3. Lighthouse (mobile) ≥ 90 performance, 100 accessibility, 100 best practices, 100 SEO on `/`.
4. `npx asciilab photo.jpg` works on a clean machine with Node 22; `npm i @asciilab/engine`
   renders in a Node script and in a Vite app.
5. README with screenshot, 30-second usage for web / CLI / library, feature matrix, and a
   comparison row versus chafa and Asciinova.
6. Privacy page live; CSP headers verified with a header check.
7. OG image renders correctly in a Twitter/X and Slack unfurl test.
8. Keyboard shortcuts dialog complete; axe scan clean.
9. Issue templates, CONTRIBUTING, CODE_OF_CONDUCT, SECURITY present; first release `v1.0.0`
   tagged with a changelog.
10. A 10-image manual QA pass (photo, logo, line art, dark scene, transparent PNG, tiny icon,
    panorama, portrait, high-contrast text screenshot, noisy JPEG) with every mode produces
    output a reviewer judges "good" and exports match the preview.

## 9. Open decisions already made (so implementers don't re-ask)

- Product and package name: `asciilab`; scope `@asciilab/*` for the engine. Rename is a
  find-and-replace before first publish.
- Default mode `tone`, default columns 120 (web) / terminal width (CLI), default dither `none`
  for stills (dither off by default because many users want clean output; one click enables).
- Default preview font JetBrains Mono; aspect auto-measured.
- Sextant blocks are offered but labelled "needs a font with Legacy Computing symbols".
- Light/dark theme follows system by default.
- No telemetry beyond Cloudflare Web Analytics page views.
