# Windows ARM64 Build Log

Tracking the effort to compile RustDesk for native Windows on ARM64 (`aarch64-pc-windows-msvc`), in response to upstream discussion https://github.com/rustdesk/rustdesk/discussions/5532.

Fork: https://github.com/johanclawson/rustdesk
Branch: `feat/windows-arm64-ci`

## Goal

Produce a working `rustdesk.exe` for Windows ARM64 via a self-contained GitHub Actions workflow that uses the free `windows-11-arm` hosted runner (publicly available since April 2025).

## Strategy

Add a **separate** workflow file rather than modifying upstream's `flutter-build.yml`, so:
- The diff against upstream is minimal and easy to review.
- The existing x64 release pipeline keeps working unchanged.
- A future PR back to `rustdesk/rustdesk` can be the single new file + small `build.py`/`.cargo/config.toml` deltas.

## Pre-build audit findings

| Item | Upstream value | ARM64 change needed |
|------|----------------|---------------------|
| Runner | `windows-2022` (x64) | `windows-11-arm` (native ARM64) |
| Rust target | `x86_64-pc-windows-msvc` | `aarch64-pc-windows-msvc` |
| vcpkg triplet | `x64-windows-static` | `arm64-windows-static` |
| Custom Flutter engine | `windows-x64-release.zip` from `rustdesk/engine` | **Not available for ARM64** — fall back to stock Flutter engine |
| `.cargo/config.toml` | has `x86_64-pc-windows-msvc` rustflags | Need matching `aarch64-pc-windows-msvc` block for `+crt-static` |
| `build.py` line 20 | hard-codes `build/windows/x64/runner/Release/` | Detect arch from env, branch to `arm64` subdir |
| Third-party `RustDeskTempTopMostWindow` | x64 only | **Skip** for first attempt |
| MSI build | uses `Platform=x64` MSBuild | **Skip** for first attempt |
| Signing | requires `SIGN_BASE_URL` secret | **Skip** (we have no cert) |
| `hwcodec` + `vram` features | x64-tuned (nasm asm, vendor SDKs) | **Drop initially**, add back once base build works |

## Iteration log

Each attempt below is a commit on this branch. The goal of each iteration is to advance past one specific failure.

### Attempt 1 — Initial workflow

- Date: 2026-05-27
- Commit: pending
- Workflow file: `.github/workflows/flutter-build-windows-arm64.yml`
- Notes: First end-to-end attempt. Uses `windows-11-arm` runner, stock Flutter engine, no `hwcodec`/`vram` features, no MSI, no signing. Expected to fail somewhere — the goal is to capture *where* and iterate.

(Subsequent attempts will be appended here.)
