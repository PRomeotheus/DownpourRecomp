# DownpourRecomp — performance and revival plan (fork of LittleBitUA/DownpourRecomp)

Upstream went quiet after v1.1.8 (2026-09-03). This fork picks it up. Its first
goal is the community complaint that the port runs worse than Xenia.

## Why it can be slower than Xenia

The CPU code is recompiled, but every pixel still goes through the same Xenia
Xenos GPU emulation. On top of that the port ships heavier defaults and its own
problems:

| # | Cause | Where | Status |
|---|---|---|---|
| 1 | Shipped defaults: `resolution_scale = 2` (4× pixels) on the ROV path, where each pixel shader also emulates EDRAM, plus FSR3, FXAA and 16× AF | SDK cvar defaults + closed-source launcher | measure first (bench presets below) |
| 2 | Two clocks: a guest vblank timer at 120 Hz, and presents on a host-clock 60 Hz timer with tearing allowed, never synced to the display. This causes judder on fixed-refresh screens | `d3d12_presenter.cpp` | **patched**: `d3d12_present_pacing` |
| 3 | The memexport readback ring advanced per draw, so a third use of one destination in a frame drained the GPU mid-frame and gave the guest another draw's data | `IssueDraw_MemexportReadbackFastPath` | **patched** |
| 4 | Readback size-cap eviction could release buffers still in flight (GPU use-after-free) | `EvictOldReadbackBuffers` | **patched** |
| 5 | First-use memexport keys each do a full `AwaitAll`: 450–525 waits in one frame at scene transitions (the 250–360 ms hitches) | full path | next: batch into one wait |
| 6 | `readback_memexport` is on for every range, though only some ranges are read by the CPU | `command_processor.cpp` | next: find the ranges the CPU reads, restrict readback |
| 7 | AMD is pinned to RTV (≈36 fps vs 58 on ROV in upstream's measurements) | launcher | goes away with ReGPU (below) |

Not fixed yet, same bug class: the resolve readback path (`readback_resolve`)
also advances its ring per call. In fast mode it reads a slot without checking
the fence. The Vulkan memexport path uses 2 slots and still has the per-call
advance.

## Phase 0 — benchmark protocol (do this before trusting any fix)

Use the same save, the same 60-second route, and PresentMon (`--output_file`).
Compare avg, 1% low and frame-time stdev.

| Run | Settings |
|---|---|
| A — Xenia Canary | `draw_resolution_scale = 1`, defaults |
| B — upstream v1.1.8 | as shipped |
| C — upstream v1.1.8 | `resolution_scale = 1`, `present_effect = 'bilinear'`, `swap_post_effect = 'none'` (like for like with A) |
| D — this fork | as C + `d3d12_present_pacing = 'limiter'` (isolates the memexport fix) |
| E — this fork | as C + `d3d12_present_pacing = 'auto'` |
| F — this fork | as E + `d3d12_frame_latency_waitable_object = true` (latency check) |

Also note `fence_wait_submission` in the `PSO_STALL[...]` log lines for B vs D.
That count is the direct signal for fix #3.

## Phase 1 — this branch

SDK branch `perf/memexport-pacing` (`rexglue-sdk-dpour`): fixes 2–4. The fixes
are untested until they're built on Windows with the game.

## Phase 2 — move onto upstream ReXGlue v0.10.1 + ReGPU

The renderer decision is recorded in `RENDERER_DECISION.md`.

## Codegen prerequisite (game files)

`downpour_manifest.toml` codegens from `assets/default.tu1.xex`. That is the
base `default.xex` (hash `7A3D5809776EE6AB`) merged with TU1 (`default.xexp`)
by XenonRecomp's XexPatcher. The merge tool is not in the repo, so it has to be
rebuilt or replaced before the first codegen.
