# VIctory XCL

A research and development project exploring a Windows compatibility runtime for Xbox Series X|S software, using VIctory as the architectural starting point and evaluating independent ideas from XCL.

## Status

**Research scaffold only. No emulator/runtime code has been merged yet.** The immediate objective is to establish a technically sound, legally compatible development path before importing or rewriting components.

## Upstream projects

- [VIctory](https://github.com/Marvisak/VIctory) — primary architectural reference and intended base.
- [XCL](https://github.com/xcl-dev/xcl) — secondary technical reference for RDNA2 shader analysis/recompilation, runtime coverage, and diagnostics.

This repository is an independent workspace. It does not modify either upstream repository.

## Current technical priorities

1. Validate Xbox-native shader container parsing and stage metadata.
2. Improve RDNA2 instruction coverage and shader translation diagnostics.
3. Validate D3D12 resource, descriptor, synchronization, and presentation behavior.
4. Verify texture layouts, swizzling, and BCPack/DirectStorage paths.
5. Build a reproducible test suite using legally obtained user-provided game data.

## Legal and asset policy

The upstream projects have different GPL licensing terms (VIctory: GPL-3.0-or-later; XCL: GPL-2.0-only, as currently identified during the initial review). Their source code must not be combined unless the licensing situation is resolved. This project will initially use independent implementations and public technical specifications, with provenance and license review for every third-party component.

No game files, copyrighted game assets, decryption keys, or DRM-bypass functionality belong in this repository. Testing must use files the tester is legally entitled to use.

## Development principles

- Keep upstream repositories unchanged.
- Work in small, reviewable commits.
- Add tests alongside each new parser or runtime behavior.
- Do not claim compatibility until it is demonstrated by reproducible tests on real hardware.
