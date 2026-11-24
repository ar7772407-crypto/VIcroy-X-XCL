# Local validation report

**Date:** 2026-11-24  
**Environment:** User-reported Windows PowerShell session, using the extracted `VIctory-main` and `xcl-main` folders.  
**Scope:** Build/test validation and a read-only shader-package inventory. No GTA VI gameplay test, shader translation-success claim, or GPU compatibility claim was made.

## VIctory

| Command | Reported result |
|---|---|
| `cargo check --workspace` | Completed successfully |
| `cargo test --workspace` | Reported successful by the user; the final `0 passed` line came from a target with no tests, while passing test summaries appeared earlier in the output |

The exact test counts and the tested source commit are not available in the conversation. The extracted ZIP does not provide a confirmed Git commit SHA.

## XCL

| Command | Reported result |
|---|---|
| `cargo check --workspace` | Completed successfully; two deprecation warnings |
| `cargo build --workspace` | Run to build the E2E test executables |
| `cargo test -p xcl --lib` | 727 passed, 0 failed, 21 ignored |
| `cargo test -p xcl --test e2e` | 3 passed, 0 failed, 0 ignored |
| `cargo test --workspace` | Final visible section: 6 passed, 0 failed, 12 ignored; the subsequent `main.rs` and doc-test targets each contained 0 tests. The individual library and E2E suites above also passed. |

### Resolved test-run issues

- The first E2E run could not find `xcl_testtitle.exe` and `testgame_mw.dll`; after `cargo build --workspace`, the E2E suite passed 3/3.
- One initial library-test run exited abnormally with Windows status `0xc0000005`. The named test passed when run alone with `--nocapture`, and the full library suite then passed 727/727. The original abnormal exit was not diagnosed, so it should be treated as a transient/unresolved test-run issue rather than a proven code defect or a proven fix.
- Two non-fatal deprecation warnings remain: `Atomic<u64>::fetch_update` in `crates/xcl/src/hle/d3d12x/real.rs` and `Atomic<u32>::fetch_update` in `crates/xcl/src/hle/xgameruntime/mod.rs`. They do not prevent compilation or the passing test runs.


## GTA VI shader-package inventory (user-run)

**Input path (kept local; not uploaded):** `shaders.rpf` from the user's configured game folder.  
**Command:** `cargo run -p xcl -- shaders "<game-folder>\\shaders.rpf" --top 50 --json`  
**Reported scan time:** 34.2 seconds.

| Metric | Result |
|---|---:|
| RPF9 package size | 392.6 MB |
| Shader records | 26,503 |
| FSSM records / programs | 26,698 |
| Xbox-native shader records | 26,503 |
| RDNA2 instructions decoded | 44,422,713 |
| Distinct mnemonics | 452 |
| Undecoded dwords | 0 |
| Inflate errors | 0 |
| Parse problems | 0 |
| Unusual records | 0 |
| Embedded data blocks | 1,473 |

DXIL-stage counts: PS 12,101; MS 9,282; CS 2,689; VS 2,236; GS 195. Hardware-stage counts also included 195 ES entries paired with GS+VS.

**Important interpretation:** this was a format/ISA inventory scan, not a recompiler run. The JSON explicitly reported `"recomp": null`, because the `--recomp` option was not supplied. Therefore this output does not establish how many of the 26,503 shaders XCL can translate, nor whether VIctory can translate them. The mnemonic order/coverage values are corpus frequency statistics, not translation-success rates.

VIctory's `shader-info` command skipped `anim.rpf` (4.1 GB) and `audio.rpf` (26.58 GB) because its current scanner has a 2 GiB per-file limit. The separately inventoried `shaders.rpf` is 0.38 GB and was scanned successfully by XCL.

### Next bounded experiment

Use XCL's direct-DXIL translator on its default seeded sample (200 programs) with validation, rather than running the full HLSL recompiler over every record immediately:

```powershell
cargo run -p xcl -- shaders "C:\\path\\to\\game\\shaders.rpf" --direct --sample 200 --validate --json --threads 4
```

This tests a sample only and is separate from the `--recomp` HLSL recompiler path. Preserve the JSON output. It is a diagnostic, not a game execution test.

## What these results establish

- Both extracted workspaces passed their reported Cargo check.
- XCL's library and E2E suites passed in repeat runs.
- The XCL test-title executable briefly opened and displayed an FPS counter during the E2E test. This is a test application, not evidence that GTA VI runs.
- No runtime source files were copied or merged between VIctory and XCL.
- No GPU render test or GTA VI shader-compatibility test has yet been performed.

## Next engineering step

Use VIctory's own diagnostic commands before changing its shader pipeline:

1. From `VIctory-main`, run `cargo run -p victory -- config` and verify that `game_dir` points to the intended local game-data folder.
2. Once the path is configured, run `cargo run -p victory -- shader-info` to inventory shader containers and native stages.
3. Then run `cargo run -p victory -- translate --validate --list --jobs 4` and preserve the output. This reports the exact programs VIctory translates or rejects and can validate emitted DXIL.
4. Compare those specific failures with XCL's documented capabilities. Do not copy XCL source into VIctory: the upstream repositories declare GPL-3.0 and GPL-2.0 licensing respectively, so code-level reuse requires a resolved licensing path.

VIctory's current README warns that entering the world on a real GPU can hang the GPU/driver. Do not repeatedly launch the world as a test until a controlled diagnostic path is in place. A successful Cargo build/test is not a GTA VI playability result.
