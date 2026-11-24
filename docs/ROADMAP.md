# Technical roadmap

## Phase 0 — Baseline and provenance

- [x] Identify VIctory as the architectural starting point.
- [x] Identify XCL as a technical reference.
- [x] Record the upstream license mismatch.
- [x] Record the user's local Cargo validation results in [LOCAL_VALIDATION.md](LOCAL_VALIDATION.md).
- [x] Review the current upstream shader/recompiler paths and CLI diagnostics.
- [ ] Verify the exact Git commit IDs of the ZIP snapshots on the user's PC, if their folders contain Git metadata.
- [ ] Establish host hardware and driver details before any GPU execution test.

The current GitHub project is a documentation/research scaffold. It does **not** contain a fork of VIctory or a merged runtime. The original upstream repositories remain unchanged.

## Phase 1 — Shader pipeline audit

### VIctory baseline
- DXBC/DXIL container parsing and Xbox-specific metadata extraction.
- GFX10 instruction decoding and translation to DXIL.
- Shader IR/control-flow analysis and host shader emission.
- Translation and pipeline-state caches.
- CLI diagnostics: `shader-info` and `translate` (including per-program listing, DXIL validation, and optional pipeline creation).

### XCL reference areas to evaluate independently
- Xbox-native XDAT/XBCH/FSSM container parsing.
- RDNA2 instruction decoding and explicit unsupported-opcode accounting.
- Wave execution semantics, SGPR/VGPR mapping, EXEC masks, LDS, image/buffer operations.
- Recompiler coverage reports and small-sample DXC validation.

### Current finding

VIctory already has a native shader parser, GFX10 decoder, IR builder, DXIL emitter, and a translation diagnostic command. XCL's recompiler is a separate and more extensive implementation, but XCL reports coverage against its own shader corpus. That coverage does not prove that the same implementation can be inserted into VIctory or will fix GTA VI.

The next comparison must be based on the *specific failed programs* reported by VIctory for the user's legally obtained target data. Compare those failed instruction semantics with XCL's documented capabilities and write a narrow, independent improvement only when a concrete gap is established.

### Required acceptance tests

- Malformed and truncated shader containers fail safely.
- Every accepted instruction has a semantic test or an explicit unsupported result.
- Generated shaders compile with the DXIL validator.
- GPU output is checked against known expected results for small deterministic tests.
- Runtime resource bindings, stage interfaces, and synchronization are validated separately from translation.
- No large corpus run is required for routine development.

## Phase 2 — GPU/runtime audit

- D3D12 device and queue lifecycle.
- Resource allocation and console-to-host memory mapping.
- Descriptor heaps, root signatures, barriers, and fences.
- Texture swizzle/layout and mip/array handling.
- Swap chain and frame presentation.
- Device removal and crash diagnostics.

## Phase 3 — Storage and formats

- DirectStorage request ordering and completion semantics.
- BCPack decoding and BC format validation.
- Texture upload/readback tests.
- Bounds checking and fuzz tests for binary parsers.

## Phase 4 — Integration

- Integrate only independently implemented and license-compatible modules.
- Keep a clear boundary between platform compatibility code and game-specific metadata.
- Add end-to-end tests using user-provided, legally obtained game files.
- Publish measured compatibility status and known limitations.

## Current next action

Run VIctory's built-in diagnostics on a known local shader-data folder before changing code:

1. `cargo run -p victory -- config` — inspect the resolved configuration and configured game-data path.
2. `cargo run -p victory -- shader-info "<path-to-your-local-shader-data>"` — inventory shader containers and native stages.
3. `cargo run -p victory -- translate "<path-to-your-local-shader-data>" --validate --list --jobs 4` — list per-program outcomes and validate generated DXIL.

Use only data the user is legally entitled to use. Do not upload proprietary game data or shader binaries to GitHub. VIctory's upstream README warns that entering the world on a real GPU can hang the GPU/driver, so do not repeatedly launch the world to investigate shader issues.

## Licensing boundary

The upstream projects declare different GPL versions (VIctory: GPL-3.0; XCL: GPL-2.0). Do not copy XCL implementation files into VIctory or combine code until the applicable licensing and provenance questions are resolved. Prefer independent implementations based on public technical descriptions, with provenance recorded for every third-party component.
