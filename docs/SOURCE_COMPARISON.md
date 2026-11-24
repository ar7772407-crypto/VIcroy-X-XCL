# Source comparison: VIctory and XCL

**Reviewed inputs:** the two ZIP archives supplied in this conversation (`VIctory-main.zip` and `xcl-main.zip`). This report describes those snapshots, not necessarily the repositories' latest commits.

## Executive result

VIctory already contains substantial functionality that would be easy to mistakenly duplicate: an XDAT/XBCH/FSSM native shader parser, GFX10 instruction decoding, an IR/control-flow builder, DXIL emission, resource/interface analysis, texture detiling, BCPack decoding, DirectStorage HLE, shader caching, and pipeline caching.

XCL's strongest potentially useful contribution is not simply "a shader translator". Its handoff describes a second observed console-shader container layout, a separate RDNA2 decoder/recompiler with measured corpus coverage, a more extensive runtime/D3D12.X implementation, and diagnostic workflows. The correct next step is a measured format/feature gap analysis against GTA VI captures, not copying XCL's recompiler wholesale.

No game executable or shader pack was included in the uploaded archives, so target-specific compatibility cannot be established from these two source trees alone.

## 1. Important existing VIctory implementation

The supplied VIctory tree already has:

- `victory-shader::native`: parses XDAT entries, inflates XBCH v3, parses FSSM v0x101, extracts per-stage native code, resource/root-layout metadata, wave size and stage IO metadata.
- `victory-shader::gfx10`: GFX10 instruction boundaries, encoding/operand decoding and SP3-style disassembly.
- `victory-shader::ir`: builds a control-flow representation from machine code.
- `victory-shader::emit`: emits host DXIL containers, with lowering, resource bindings, wave operations, host validation and tests.
- `victory-gpu::texture`: updates host textures from console-tiled memory, currently with documented limitations (whole-texture detiling after a change, descriptor-table views only, plane 0, no depth/stencil planes).
- `victory-formats::texture`: GFX10.3 texture layout and tiled/linear address calculations.
- `victory-formats::bcpack`: BCPack decoder for BC1–BC7 (BC2 intentionally omitted because the title data observed by VIctory did not use it).
- `victory-hle::modules::dstorage`: asynchronous DirectStorage queues, file/memory sources, zlib and BCPack unpacking, and completion/status handling.
- GPU shader and pipeline caches in `victory-gpu` / `victory-pipeline`.

These are real implementations in the uploaded VIctory source, not proposed features.

## 2. Material difference in native shader container layouts

The two projects document different FSSM layouts.

### VIctory snapshot

`crates/victory-shader/src/native.rs` expects the FSSM payload to begin with a graphics-slot count and a table of stage-record offsets. Each stage record contains code offset/size, flags, root-layout and used-resource tables, and register metadata. VIctory also reads stage-specific IO metadata (PS inputs/export patches, VS inputs/outputs, CS dimensions, MS data).

### XCL snapshot

`crates/xcl/src/gpu/xbox.rs` and `HANDOFF.md` describe a different FSSM organization: code offset/size at fixed header offsets, hardware-stage descriptors at a fixed table, and program/data-block discovery around `s_endpgm` markers. XCL's notes describe 23,869/26,503 shader-pack samples depending on the handoff section/version and report parsing all samples in the referenced corpus.

### Consequence

Do not replace VIctory's parser with XCL's parser. First add a format discriminator and test corpus proving whether the layouts are distinct title/compiler variants, or whether one is a partial/older observation. If GTA VI supplies the VIctory layout, the XCL layout may be irrelevant to the current rendering failure. If GTA VI supplies both, VIctory needs a second parser path that preserves stage and resource metadata as far as the format exposes it.

## 3. Recompiler comparison

XCL's `HANDOFF.md` describes a dedicated RDNA2 → HLSL recompiler, including SALU/VALU, SDWA/DPP, memory and image operations, atomics, LDS, wave operations, mesh mode, and DXC compilation. It reports corpus-based full-translation coverage in successive handoff milestones and tests on its own retail shader pack and GPU.

VIctory has its own GFX10 decoder, IR builder and DXIL emitter. Therefore the actionable comparison is opcode and semantic coverage, not source-code transplantation.

Required comparison procedure:

1. Extract the actual unsupported-opcode report from VIctory's `apps/victory translate` output for GTA VI shaders.
2. Normalize instruction names and encodings against the XCL recompiler's documented supported set.
3. For each gap, determine whether it is a missing decode, IR operation, semantic lowering, resource-binding rule, or host-DXIL limitation.
4. Implement a narrow independent fix in VIctory with a minimal synthetic test and a captured-shader regression test.
5. Run DXC/host validation and, where possible, a GPU execution test before counting it as supported.

A translator reporting "complete" is not enough by itself: the resource bindings, stage interfaces, memory model, and runtime pipeline state must also match.

## 4. Runtime and resource-path opportunities

XCL's source tree contains additional implementation areas worth comparing against VIctory's corresponding code:

| Area | VIctory snapshot | XCL snapshot | Recommended investigation |
|---|---|---|---|
| Shader container | `victory-shader/native.rs` | `gpu/xbox.rs` | Determine whether GTA VI uses more than one FSSM layout |
| RDNA2 recompiler | `gfx10`, `ir`, `emit` | `gpu/rdna2.rs`, `gpu/recomp/*` | Build an opcode/semantic coverage diff from real captures |
| D3D12.X | `victory-gpu/*`, `hle/modules/d3d12x.rs` | `hle/d3d12x/*` | Compare exact interface slots, PSO creation, binding and synchronization |
| UMA / console memory | `victory-gpu/resource.rs` and HLE memory | `gpu/uma.rs`, `gpu/vram.rs` and D3D12.X memory modules | Check GPU visibility and aliasing for captured resources |
| Texture updates | `victory-gpu/texture.rs` | `gpu/texture.rs`, `hle/d3d12x/title_textures.rs` | Compare direct-indexed descriptors, depth/stencil and update granularity |
| Texture layout | `victory-formats/texture.rs` | `gpu/tiling.rs`, `gpu/tiling_tables.rs` | Validate pipe/bank/swizzle modes against actual game resources |
| BCPack / DirectStorage | `victory-formats/bcpack.rs`, `hle/modules/dstorage.rs` | `hle/bcpack/*`, `hle/dstorage.rs` | Compare edge cases and corpus tests; avoid duplicating already working paths |
| Diagnostics | VIctory translation/capture tools | `xcl shaders`, `xcl disasm`, reports | Add a reproducible per-shader failure report to VIctory |

XCL's latest handoff also describes title-specific issues discovered only during runtime: large memory pressure, tiled VRAM aliasing, format reinterpretation, descriptor/resource binding, command-list methods, pipeline creation, and presentation behavior. These are useful categories for investigation, but they are not evidence that the same fixes apply to GTA VI.

## 5. Why the dark scene is not yet attributable to one cause

VIctory's README reports that GTA VI reaches the first mission intro, the title card renders, and the following scene is dark; indirect draws are reached, but performance is extremely slow. The same README warns that running the world on a real GPU can hang the GPU/driver.

Missing shader translations are a plausible cause, but the source comparison also identifies other plausible failure classes:

- incorrect native shader-container variant or stage metadata;
- incomplete instruction semantics or control-flow lowering;
- incorrect root/user-data to SGPR mapping;
- descriptor heap addressing or directly indexed resource visibility;
- texture swizzle/pipe mode, depth/stencil plane or format mismatch;
- resource state/barrier/fence ordering;
- command-list or indirect-draw semantics;
- PSO creation or host pipeline incompatibility.

The first useful experiment is to capture a failing frame's PSOs, shaders, resource descriptors, command-list operations and debug-layer messages, then correlate the exact missing/invalid item with the draws that fail. Avoid repeatedly entering the world on the real GPU until the hang risk is addressed; use the diagnostic/software path where feasible.

## 6. Licensing boundary

The supplied VIctory tree declares **GPL-3.0-or-later**. The supplied XCL tree declares **GPL-2.0-only** and its handoff explicitly identifies third-party provenance for some recompiler work. GPL-2.0-only code cannot simply be copied into a GPL-3.0-or-later codebase as though the licenses were interchangeable.

For this project:

- do not copy XCL implementation files into VIctory;
- do not port third-party-derived logic without reviewing its provenance and license;
- use public technical descriptions and independently written implementations only where legally appropriate;
- record provenance and license for every future third-party component;
- ask the relevant copyright holders or obtain legal review before attempting code-level combination.

This is a project constraint, not a technical inconvenience to work around.

## 7. Validation constraints and next engineering step

The current execution environment used for this review has no Rust toolchain (`rustc`/`cargo` are unavailable), and the supplied archives do not include the GTA VI executable or its captured shader/resource data. Consequently, no build, unit test, shader compilation, or GPU test is claimed here.

Recommended next implementation slice:

1. Run VIctory's existing shader census/translation diagnostics against a legally obtained GTA VI shader capture.
2. Export a machine-readable list of failed shaders with stage, container layout, opcode, byte offset, and failure category.
3. Compare those exact failures against VIctory's existing code and XCL's documented capabilities.
4. Select one high-frequency, well-isolated gap and implement it independently in VIctory, with a regression test.

This avoids spending time reimplementing features VIctory already has and directs effort at the actual failing draws.
