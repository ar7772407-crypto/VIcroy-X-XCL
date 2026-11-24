# VIctory XCL — integration architecture

## 1. Goal

Use VIctory as the host architecture for a research and compatibility project targeting Xbox Series X|S titles on Windows. XCL is a technical reference, not a source of code to copy blindly. The immediate goal is to improve observability and correctness in the shader/GPU path before claiming any game compatibility.

This document is a research plan, not evidence that the integration is implemented or that GTA VI runs.

## 2. Repository inputs

- VIctory: https://github.com/Marvisak/VIctory
  - Rust 2024, GPL-3.0-or-later.
  - Existing crates include loader, HLE, GPU, shader, formats, pipeline and game-specific support.
  - The current project status says GTA VI is not playable; the reported blocker includes missing shader translations and incomplete rendering.
- XCL: https://github.com/xcl-dev/xcl
  - Rust 2021, GPL-2.0-only.
  - Research notes describe a console-native shader container parser (XDAT/XBCH/FSSM), RDNA2 instruction decoder, HLSL recompiler, runtime D3D12.X layer, and shader diagnostics.
  - Its repository handoff reports strong translation coverage on its own sample corpus, but that is not proof of compatibility with GTA VI or with VIctory's runtime.

## 3. Licensing boundary

VIctory is GPL-3.0-or-later while XCL is GPL-2.0-only. Do not copy or link XCL implementation code into VIctory unless the relevant authors grant compatible licensing or the project owners explicitly resolve the licensing issue. Keep this repository's documents and independently written code separate from both upstream projects until that question is settled. Any future imported component must have a recorded source, exact license, author attribution, and compatibility review.

## 4. What should be reused conceptually

### Shader format and capture

VIctory already has a shader crate with DXBC/DXIL parsing and GFX10 translation modules. XCL's reverse-engineered console shader path provides a useful checklist for diagnostics and edge cases:

1. Identify the DXBC container and DXIL part.
2. Detect console-specific XDAT metadata rather than assuming the DXIL bitcode is ordinary PC DXIL.
3. Validate XDAT table bounds and offsets.
4. Decompress XBCH payloads safely with strict size limits.
5. Parse FSSM headers, hardware-stage descriptors, code ranges, program boundaries, and embedded data blocks.
6. Preserve unknown metadata and report parse fallbacks instead of silently dropping a shader.

Before implementing a second parser, compare VIctory's existing `victory-shader` modules (`dxbc`, `dxil`, `native`, `gfx10`, `emit`) against these requirements and add only missing behavior.

### RDNA2 translation

The most direct technical lesson from XCL is that shader translation needs measurable coverage, not merely a successful parser. A useful independent design should report, per shader and stage:

- instruction count and undecoded dwords;
- translated program count and fully translated program count;
- unsupported opcodes and unsupported operand/control-flow forms;
- DXC compile result and compiler diagnostics;
- resource/interface metadata needed to bind the translated shader correctly.

Do not treat a generated HLSL file as a valid translation until it compiles and its execution semantics are tested. Prioritize the exact shader forms captured from the target game rather than broad opcode coverage without evidence.

### GPU/runtime boundary

Keep console address-space semantics, descriptor translation, resource formats, synchronization, and host D3D12 object ownership explicit. A translated shader can still render incorrectly if its resource bindings, memory visibility, texture layout, or pipeline state are wrong.

### Caches

Keep shader translation cache keys tied to the input shader bytes and translator/configuration version. Keep host pipeline cache validity tied to adapter, driver, and relevant pipeline state. Never reuse stale cache entries after translator semantics change.

## 5. Likely rendering investigation order

The VIctory status notes indicate that some shaders are not translated and that omitted draws can leave the scene dark. Investigate in this order:

1. **Capture evidence:** dump the exact shader blobs and PSO creation metadata from the failing scene.
2. **Classify:** distinguish ordinary DXIL/DXBC from console-native containers and unknown formats.
3. **Parse:** verify all shader stages and code ranges; count every rejected or skipped shader.
4. **Translate:** map unsupported instructions to the actual failing shaders; avoid guessing based on aggregate opcode counts.
5. **Compile:** compile each translated shader with DXC and retain diagnostics.
6. **Bind:** validate root signatures, descriptor tables, user-SGPR mapping, resource formats, and stage interfaces.
7. **Execute:** compare draw/dispatch counts, barriers, resource state transitions, and GPU validation messages.
8. **Render regression:** use small known scenes and image/pixel comparisons before testing the full game scene.

A black or dark scene is not sufficient evidence that shader translation alone is the cause.

## 6. Milestones

- [ ] Establish a reproducible baseline from the current VIctory main branch.
- [ ] Inventory the existing VIctory shader and GPU modules against the checklist above.
- [ ] Capture and classify representative target-game shaders.
- [ ] Produce a ranked unsupported-feature report based on captured target data.
- [ ] Implement one independently written, narrowly scoped improvement with unit tests.
- [ ] Validate shader compilation and GPU execution on a small reproducible test.
- [ ] Reassess licensing before any code-level reuse from XCL.

## 7. Non-goals and safety

This project does not provide game decryption, DRM circumvention, title downloads, or instructions to obtain proprietary game files. Testing assumes the user lawfully possesses any required game data. No compatibility or playability claim should be made without reproducible end-to-end evidence.
