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

Launch with `TOTK_HDR=0` for SDR. Restart to change modes: the game shader variant and layer format are selected at startup. The native launcher separates HDR's pipeline cache with an `-hdr1` suffix. Shader source and game data are not modified. The local experimental app can carry `LSEnvironment.TOTK_HDR=1` in its signed Info.plist so Finder enables HDR; this is an artifact configuration, not the default of the source tree.

`wshot` reads float drawables with an appropriately sized buffer, reports peak values/pixels above SDR white, and produces an SDR preview. That PNG is not an HDR brightness reference. There is no per-frame CPU pixel readback in ordinary play.

## Validation

The independent probe compiles and executes the production presentation shader, checks real float pixels above 1.0, unchanged diffuse gray and reference white, SDR overlay decoding, SDR output bytes, nonfinite pixels, and headroom bounds:

```sh
clang -fobjc-arc -O2 -Wall -Wextra -Werror \
  -I games/totk/third_party/d2r_gfx/gfx/nvn \
  games/totk/tests/hdr/probe.m -framework Cocoa -framework Metal -o /tmp/totk-hdr-probe
MTL_DEBUG_LAYER=1 /tmp/totk-hdr-probe
make check
```

Initial probe: 8,223 checks passed with Metal API Validation. The full repository suite passed 332 Python tests plus its C checks, including portable reference-white/headroom invariants.

Copied-save outdoor gameplay completed with HDR on, HDR off, and HDR plus MetalFX spatial/output upscaling. Native RGBA16F drawable readback measured peak values of 1.9131 SDR-white units (HDR) and 2.8203 (HDR + MetalFX), with 3,751 and 8,809 pixels above white respectively and no nonfinite components. MetalFX output upscaling from 1920×1080 to 3024×1700 measured about 1.17 ms median / 1.26 ms p95 GPU time in a short fixed-camera test; this is not a broad performance benchmark. All three sessions exited cleanly.

Full-game Metal API Validation encountered `Attachment 0 overlaps with attachment 1` during loading, both with HDR on and with HDR off, in the renderer's render-stream worker. The same error was then reproduced with the untouched upstream Mac app. The separate production-HDR-shader probe passes validation; the gameplay checks run without the full-game validation layer. This existing renderer problem is not treated as a passed validation run or fixed by this feature.

Remaining review: subjective highlight grading across different regions/weather, sustained performance and memory, many swapchain re-creations, external HDR/SDR displays and display switching. The capture intentionally accepts only the recognized final copy; unusual final passes fall back to the game's SDR swapchain.

Apple references: [EDR rendering conventions](https://developer.apple.com/videos/play/wwdc2021/10161/), [performing your own tone mapping](https://developer.apple.com/documentation/metal/performing-your-own-tone-mapping).
