# windows_canvas_rn

The whole `CanvasRenderingContext2D` API as a recording context
(`src/recorder.ts`), and a React Native Windows Composition view that replays
the recording with Direct2D, DirectWrite and Direct2D effects (`windows/`).
Beside it, `LineScene`: a Composition view that draws a 3D scene of line
segments with Direct3D 11 (`src/line-scene.ts`, `windows/LineScene.cpp`). See
README.md for the API, its one limit (`getImageData` cannot read back) and the
Windows setup.

## Commands

- `bun run verify` — format, typecheck, lint, tests, build. Before any push.
- `bun run test:unit` — vitest, node environment.

## Rules

- **The recorder resolves canvas state; the replay keeps only four things.**
  Points are transformed, alpha folded into colours, arcs turned into cubics,
  text anchored, before an op is written. The replay's only state is what a
  canvas applies to every op — composite, shadow, filter, smoothing — set by
  `Set…` ops written on change. A new feature is a format change: bump
  `FORMAT_VERSION` in `src/format.ts` and `windows/PictureFormat.h` together;
  `format.test.ts` compares them, and `decode.ts` is the reference reader the
  C++ mirrors.
- **The API is the spec's, edges included.** `recorder.test.ts` lists every
  member of `CanvasRenderingContext2D` and fails if one is missing; invalid
  values are ignored and the spec's throws throw. Check a behaviour against
  the HTML spec before changing it.
- **A layer composites source-over.** Anything that must replace pixels —
  `clearRect` under a clip, every composite but source-over — goes through an
  effect graph against a copy of the target and is written back with a clip
  mask (`Replayer::ReplaceThroughClip`), never inside a pushed layer.
- **`src/` is published and is what Metro loads** (`react-native` points at
  it): React Native's codegen Babel plugin reads the prop types in
  `WindowsCanvasPictureNativeComponent.ts`, which compiled JavaScript no longer
  has. Relative imports carry `.ts`/`.tsx` extensions so Metro finds the file
  exactly; `rewriteRelativeImportExtensions` turns them into `.js` in `dist`,
  which is what Node and vitest load.
- **`src/core.ts` must not import React Native**, directly or through anything
  it re-exports: it is the entry for tests and workers.
- **None of the native code can be compiled on a Mac.** It follows the
  patterns of react-native-svg's Windows Fabric `SvgView` and the
  `MoosiacNativeSlider` in mail_box_components_rn, which do build against React
  Native Windows 0.81; check a change against those before assuming an API.
- **One draw per transaction.** Props, layout and mounting all call
  `Invalidate`, which posts a single draw to the UI dispatcher. Drawing inside
  each callback is the react-native-svg bug this package exists to avoid.
- **`LineScene` draws with Direct3D on the surface's own device, inside its
  own `ID3DDeviceContextState`.** The device is reached through the surface
  context's target (`ID2D1Bitmap1` → DXGI surface → `ID3D11Device`) so
  Direct2D can draw the result as a shared bitmap; when that fails it draws on
  a device of its own and reads back through memory. The state object is
  swapped in for the draw and out again, so neither its pipeline state nor
  Direct2D's leaks into the other through the shared immediate context. The
  vertex buffer is rebuilt only when `vertices` arrives in `SetProp` — which
  is why `LineScene.tsx` must keep an unchanged vertex array the same array.
- **Native sources are listed in `windows/WindowsCanvas.targets`**, which apps
  import. Add a file there, not in an app project.

## Screen rendering — safe area & scroll insets

Honour the device safe area on **every** screen. Always use `useSafeAreaInsets()`
from `react-native-safe-area-context` (or a `SafeAreaView` with explicit `edges`)
and pad by the actual inset value. **Never hard-code a padding/margin in place of
an inset** — a magic number like `paddingBottom: 80` happens to clear today's
phone and is wrong on the next one.

Decide per axis, by whether that axis scrolls:

**Vertical:**
- **NOT vertically scrollable** (a fixed screen, a pinned footer/CTA, a camera
  overlay): the content area must stop at the **top and bottom** safe-area insets
  — pad top by `insets.top`, bottom by `insets.bottom`. It must not reach the
  physical top/bottom edge (status bar / notch / home indicator).
- **Vertically scrollable**: let the scroll view render **full-bleed** to the top
  and bottom edges, and give it top/bottom **scroll insets** equal to the safe
  area (`contentInset` + `scrollIndicatorInsets`, or
  `contentInsetAdjustmentBehavior="automatic"`). Content then starts/ends clear of
  the unsafe regions while still scrolling edge-to-edge behind them.

**Horizontal** (landscape, display cutout, rounded corners):
- **NOT horizontally scrollable**: pad left/right by `insets.left` / `insets.right`.
- **Horizontally scrollable**: render to the left/right edges and give it
  left/right scroll insets.

**Bottom CTAs under a tab bar — the trap that burns everyone:** inside a tab
navigator, the bottom safe-area inset (`insets.bottom`, and what a shared
`BottomActionBar` reads) **already includes the tab-bar height**. So a pinned
bottom bar padded by `insets.bottom` already lands above the tab bar. Therefore:
- DO pad the bottom bar by `insets.bottom` and nothing more.
- DON'T add the tab-bar height on top of the inset, and DON'T reserve it again on
  the screen root — that double-counts and floats the CTA too high.
- DON'T pass a `tabBarHeight` (or similar) that makes the bar swap the inset for a
  flat fixed gap — that ignores the inset and drops the CTA under the tab bar.

Canonical fixed-screen shape: a flex-column root → the content/scroll region →
a bottom action bar as the **last child** that pads by `insets.bottom` with no
manual tab-bar math. Copy a screen already doing this correctly (e.g. a Settings
screen with a Sign-Out button) rather than re-deriving the padding.
