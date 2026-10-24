# Experimental Mac HDR / EDR

This is an opt-in prototype for Tears of the Kingdom 1.4.3 on macOS. `TOTK_HDR=1` enables it; the ordinary SDR path remains the default. Initial validation uses an Apple M5 Mac's Liquid Retina XDR display. iOS and Breath of the Wild are not enabled.

## Rendering path

The original compose shader reads an exposed floating-point scene and bloom, compresses it, and clamps it to SDR. The game then applies FXAA/upscaling, draws its UI into a floating-point canvas, and copies the canvas through an sRGB view into its 8-bit swapchain.

The prototype:

1. Matches the compose shader by its full original shader hash and verifies the expected translated-source expressions. It retains the original color treatment and midtones, extending the highlight shoulder using **pre-tonemap scene luminance**. An unknown source layout keeps the original shader.
2. Matches the final copy's vertex and fragment hashes, checks its viewport and source format, and copies the completed float canvas to a separate RGBA16F texture for that swapchain slot before SDR quantization. Guest texture formats, memory layouts and CPU-visible storage are unchanged. The copy is on the existing Metal queue; capture frames must match presentation frames.
3. Presents through an RGBA16F `CAMetalLayer`, `wantsExtendedDynamicRangeContent`, and extended linear sRGB. The original sRGB swapchain path is kept as a fallback for unrecognized frames.
4. Refreshes the window screen's current EDR headroom on the main thread. Linear values pass through when below the shoulder; stronger highlights roll off smoothly within available headroom. SDR reference/UI white remains 1.0. SDR fallback frames and overlays are decoded exactly once for the linear EDR layer.
5. Uses MetalFX's HDR color-processing mode for output spatial upscaling. Frame interpolation is disabled: its existing RGBA8 sRGB output would clip highlights.

This is genuine floating-point EDR presentation with scene-guided highlight reconstruction. It is **not an absolute-nit HDR master or unchanged Nintendo HDR passthrough**: the game was authored for SDR, so highlight rendering is a new artistic choice. It also does not claim a wider original color gamut.

## Use and storage

Build the Mac app with `build_app.py` as usual, then launch it with:

```sh
open -n --env TOTK_HDR=1 '/path/to/Tears of the Kingdom HDR.app'
```

Use `TOTK_HDR_CONTROL=1` to expose **8 — HDR / EDR** in the existing shortcut overlay. Press **8** to switch live; the overlay shows the current on/off state in both compact and corner layouts, with English and Chinese labels. As with keys 1–7, holding the key does not toggle repeatedly. There is no separate Display menu. It remembers the last choice in the dedicated `org.macnativegames.totk.hdr-test` preferences suite (first launch defaults to on). Explicit `TOTK_HDR=0` or `1` overrides the initial choice for diagnostics. Turning EDR off executes the original compose shader math and presents the SDR swapchain; turning it back on restores the float canvas. The layer stays extended linear sRGB / RGBA16F to avoid changing surface formats while frames are queued. EDR display opt-in and effective headroom follow the switch. Mode is latched per render frame and bound as immutable GPU uniform data. The native launcher separates HDR's pipeline cache with an `-hdr1` suffix whenever EDR control is available, including while EDR is off. Test profiles and warmed shader caches are retained. Shader source and game data are not modified. The local experimental app can carry `LSEnvironment.TOTK_HDR_CONTROL=1` in its signed Info.plist so Finder exposes the shortcut; this is an artifact configuration, not the default of the source tree.

`wshot` reads float drawables with an appropriately sized buffer, reports peak values/pixels above SDR white, and produces an SDR preview. That PNG is not an HDR brightness reference. There is no per-frame CPU pixel readback in ordinary play.

## Validation

The independent probe compiles and executes the production presentation shader, checks real float pixels above 1.0, unchanged diffuse gray and reference white, SDR overlay decoding, SDR output bytes, nonfinite pixels, and headroom bounds:

```sh
clang -fobjc-arc -O2 -Wall -Wextra -Werror \
  -I games/totk/third_party/d2r_gfx/gfx/nvn \
  games/totk/tests/hdr/probe.m -framework Cocoa -framework Metal -o /tmp/totk-hdr-probe
MTL_DEBUG_LAYER=1 /tmp/totk-hdr-probe
make check
# Optional: exercise the exact compose MSL extracted from your own game.
clang -fobjc-arc -O2 -Wall -Wextra -Werror \
  -I games/totk/third_party/d2r_gfx/gfx/nvn \
  games/totk/tests/hdr/compose-probe.m -framework Foundation -framework Metal -o /tmp/totk-hdr-compose-probe
MTL_DEBUG_LAYER=1 /tmp/totk-hdr-compose-probe /path/to/owned-compose.metal
```

The shortcut regression probe exercises production key-8 dispatch, held-key repeat suppression, multiple queued presses, live on/off text, English and Chinese labels, both HUD layouts, and key-1 visibility. A copied-save gameplay run using key 8 completed on → off → on without restart or nonfinite pixels: peaks were 3.3594, 1.0000 (zero pixels above white), and 3.3633 respectively. EDR-off preferences were verified across a LaunchServices relaunch. In one copied-save gameplay process, drawable peaks were 3.3633 with EDR on, 1.0000 with EDR off (zero pixels above white), and 3.3867 after re-enabling, with no nonfinite components. Both the normal profile and the existing HDR test profile remained byte-for-byte unchanged.

The live-control compose probe verifies disabled output is byte-identical to the original SDR shader before and after enabling HDR; the synthetic highlight reaches 3.4492 in the HDR case. Unknown or already patched source is rejected.

Initial probe: 8,223 checks passed with Metal API Validation. The full repository suite passed 332 Python tests plus its C checks, including portable reference-white/headroom invariants.

Copied-save outdoor gameplay completed with HDR on, HDR off, and HDR plus MetalFX spatial/output upscaling. Native RGBA16F drawable readback measured peak values of 1.9131 SDR-white units (HDR) and 2.8203 (HDR + MetalFX), with 3,751 and 8,809 pixels above white respectively and no nonfinite components. MetalFX output upscaling from 1920×1080 to 3024×1700 measured about 1.17 ms median / 1.26 ms p95 GPU time in a short fixed-camera test; this is not a broad performance benchmark. All three sessions exited cleanly.

Full-game Metal API Validation encountered `Attachment 0 overlaps with attachment 1` during loading, both with HDR on and with HDR off, in the renderer's render-stream worker. The same error was then reproduced with the untouched upstream Mac app. The separate production-HDR-shader probe passes validation; the gameplay checks run without the full-game validation layer. This existing renderer problem is not treated as a passed validation run or fixed by this feature.

Remaining review: subjective highlight grading across different regions/weather, sustained performance and memory, many swapchain re-creations, external HDR/SDR displays and display switching. The capture intentionally accepts only the recognized final copy; unusual final passes fall back to the game's SDR swapchain.

Apple references: [EDR rendering conventions](https://developer.apple.com/videos/play/wwdc2021/10161/), [performing your own tone mapping](https://developer.apple.com/documentation/metal/performing-your-own-tone-mapping).
