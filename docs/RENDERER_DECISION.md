# Renderer: finish the archived native renderer, or move to ReGPU?

**Decision: ReGPU first. Keep the native-renderer research as a reference, not as code to finish.**

## The archived native renderer (upstream commits up to b480f9e)

What it had:
- Strong reverse engineering: the Xenos register shadow offsets in the D3D
  device, the RHI function addresses, the UP-draw/Begin/End layout, inverted
  depth, and the colour-grading LUT path.
- 419/419 shaders translated.
- Replay of the title screen and UI.

All of that is in `NATIVE_RENDER_HANDOFF.md` and the git history.

Where it stopped:
- The upstream author's own last measurement: **49.5 ms median vs 16.6 ms on
  the emulated path (3× slower)**. The cause is reading vertex/index buffers,
  textures and constant banks back out of guest memory on every draw. Avoiding
  that cost means owning every resource's lifetime through the UE3 RHI.
  UnleashedRecomp does exactly that, and it is the unbuilt hard part.
- The menu was still missing its background, menu items and prompts. Depth
  SRVs, depth-only passes and RHIClear were not routed. Gameplay was never
  reached.
- It was a game-specific hybrid that hooked device entry points. The author
  retired it on 2026-08-08.

Finishing it means months of work on one game.

## ReGPU (zer0ducksgiven/ReGPU, MIT; also packaged as Re360's "XboxGPU")

What it is:
- A ReXGlue GPU plugin. It reads the same PM4 stream, but keeps render targets
  as host textures and turns resolves into copies.
- No EDRAM/ROV emulation, so the biggest GPU cost and the 7e3 resolve artefact
  class go away.
- Shaders become readable HLSL. FSR/DLSS are built in.
- It is game-neutral, with hooks for per-game fixes (`rewriteShader`, `onDraw`,
  `resolveWriteback`).
- It already runs other games, and SSX (2012) ships it as the default.

Blocking gap for Downpour: **"No tessellation; memexport writes are dropped."**
Downpour's HUD/subtitle/journal text depends on memexport, and upstream's notes
say skinning and indirect draw counts may too. Two things are needed:
1. Translate `eA`/`eM0-4` into `RWByteAddressBuffer` writes against a guest
   memory mirror, with readback of the written ranges.
2. A frame-batched sync like fix #5.

That is generic, upstreamable work, much smaller than a game-specific renderer.
UE3 has not been tested on ReGPU yet, so expect title-specific bugs.

## Order

1. Ship Phase 1 on the existing emulated path (fast to verify, keeps a fallback).
2. Port the project to the upstream v0.10.1 SDK in the RexGlue workspace,
   carrying the Downpour SDK patches that are worth keeping.
3. Add memexport to ReGPU, then boot Downpour on `--gpu_plugin=regpu`. Keep
   `xenos` selectable as a fallback.
4. Use the native-renderer RE for game hooks (DoF kill, LUT, FOV/ultrawide), not
   for drawing.
