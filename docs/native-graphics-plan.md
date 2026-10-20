# Native graphics controls for TOTK 1.4.3

This is a proposed implementation plan. No new graphics controls or rendering
changes are implemented by this document.

The aim is to replace the game's adaptive image-quality compromises with
explicit controls suited to the Mac. The real-time hair and cloth correction
is maintained independently on the
[clothes branch](https://github.com/ChiefSZ13/MacNativeGames/tree/codex/totk-native-graphics),
with its own timing documentation and regression checks.

## Rendering path

The intended path is a scene rendered at a chosen internal resolution, followed
by MetalFX upscaling when needed, then HUD composition and presentation. Internal
scene resolution and output resolution must be independently selectable.

Initial work:

1. Add a fixed internal-resolution option, initially testing 1920x1080. Extend
   the existing `TOTK_RENDER_RES` hook and verify the effective scene viewport
   and render targets remain fixed under load; a configured allocation size
   alone does not prove dynamic resolution is disabled.
2. Replace the game's built-in FSR pass with MetalFX when upscaling. Provide a
   valid copy path when scene and output resolutions match. The existing
   MetalFX interception is a starting point; it currently has eligibility
   checks and can fall back to the game's original pass.
3. Expose FXAA independently and compare identical scenes with it enabled and
   disabled. Check both image sharpness and aliasing in motion.
4. Make these choices persistent and accessible from the macOS menu bar.

Investigate adaptive quality reduction and LOD separately. Identify the actual
1.4.3 quality decisions before overriding them. Tune detail distances with
measured CPU, GPU and memory costs; MetalFX cannot recover geometry or effects
that were omitted before rendering.

## Reference and integration

[nx-optimizer](https://github.com/MaxLastBreath/nx-optimizer/tree/main/src)
is a reference for available controls. Its
[TOTK options](https://github.com/MaxLastBreath/nx-optimizer/blob/main/src/PatchInfo/Tears%20Of%20The%20Kingdom/Options.json)
include dynamic resolution, FSR, FXAA, render distance and forced LOD. Its
[compatibility list](https://github.com/MaxLastBreath/nx-optimizer/blob/main/src/PatchInfo/Tears%20Of%20The%20Kingdom/PatchInfo.json)
includes 1.4.3. The Python configuration layer writes UltraCam settings; the
native runtime still needs corresponding, version-checked hooks. No general
adaptive-quality switch was identified in the inspected options.

Relevant local code:

- `games/totk/runtime/game_patches_v143.c`: guarded scene-size patch and
  existing real-time frame-step integration.
- `games/totk/third_party/d2r_gfx/gfx/nvn/nvn_mtl.m`: built-in upscale-pass
  interception and MetalFX integration, including the 1.4.3 shader variant.
- `games/totk/third_party/d2r_gfx/gfx/nvn/totk_mfx_up.m`: MetalFX scaler setup.

## Validation and data preservation

Compare the same saved scene and camera position, changing one setting at a
time. Inspect actual scene sizes, frame times, HUD placement, color, aliasing
and fallback behavior. Exercise heavy scenes before choosing defaults.

Retain existing shader caches and normal saves. Newly exercised shader variants
may compile in addition to the retained cache. Use copied profiles for gameplay
validation and keep generated code, game inputs, logs and build outputs outside
the source repository. When integrating with the clothes branch, retain and
run its hair/cloth timing regression checks alongside graphics validation.
