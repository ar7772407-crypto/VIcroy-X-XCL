# Technical roadmap

## Phase 0 — Baseline and provenance

- [x] Identify VIctory as the architectural starting point.
- [x] Identify XCL as a technical reference.
- [x] Record the upstream license mismatch.
- [ ] Pin exact upstream commit SHAs and license files before any code is considered for reuse.
- [ ] Establish build/test environment and supported host hardware.

## Phase 1 — Shader pipeline audit

### VIctory baseline
- DXBC/DXIL container parsing and Xbox-specific metadata extraction.
- GFX10/RDNA2 instruction decoding and translation.
- Host shader emission and DXIL compilation.
- Shader translation cache and pipeline-state cache.

### XCL reference areas to evaluate independently
- Xbox-native XDAT/XBCH/FSSM container analysis.
- RDNA2 instruction coverage reports and unsupported-opcode accounting.
- Wave execution semantics, SGPR/VGPR mapping, EXEC masks, LDS, image/buffer operations.
- Shader corpus tests and small-sample DXC validation.

### Required acceptance tests
- Malformed and truncated containers fail safely.
- Every accepted instruction has a semantic test or an explicit unsupported result.
- Generated shaders compile on the target D3D12 driver.
- GPU output is checked against known expected results for small deterministic tests.
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
- Bounds checking and fuzz tests for all binary parsers.

## Phase 4 — Integration

- Integrate only independently implemented and license-compatible modules.
- Keep a clear boundary between platform compatibility code and game-specific metadata.
- Add end-to-end tests using user-provided, legally obtained game files.
- Publish measured compatibility status and known limitations.

## Current blocker

This connected GitHub workspace currently contains an empty repository. The GitHub connector does not expose a fork operation, so upstream source cannot be safely forked into this workspace automatically. The original upstream repositories will remain untouched. A GitHub fork of VIctory (or an archive/working copy supplied by the project owner) is needed before modifying the actual runtime source.
