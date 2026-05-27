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
- Run: https://github.com/johanclawson/rustdesk/actions/runs/26493736986
- Result: ❌ failed at "Install Flutter" step.
- Error: `Unable to determine Flutter version for channel: stable version: 3.24.5 architecture: arm64`
- Diagnosis: `subosito/flutter-action` auto-detects `RUNNER_ARCH=ARM64` on the `windows-11-arm` runner and tries to download an ARM64 Flutter SDK. **Flutter does not publish a Windows-on-ARM SDK** — the official releases JSON (`storage.googleapis.com/flutter_infra_release/releases/releases_windows.json`) contains *zero* `dart_sdk_arch=arm64` entries for any channel. The x64 SDK runs fine on Windows-11-ARM under emulation; the lack of an ARM64 SDK only means we cannot host-build at native speed, not that we cannot cross-build ARM64 apps.
- Bridge job: ✅ succeeded (so the bridge.yml dependency works).

### Attempt 2 — pin Flutter 3.32.8, x64 SDK, ARM64 cross-build

- Plan:
  1. Bump `FLUTTER_VERSION` to **3.32.8** (Dart 3.8.1 — has stable Windows-ARM64 cross-build target since 3.32).
  2. Pass `architecture: x64` explicitly to `subosito/flutter-action` so it stops trying to find an ARM64 SDK.
  3. Drop the `flutter_3.24.4_dropdown_menu_enableFilter.diff` patch step — that diff targets the 3.24.x dropdown source and won't apply cleanly on 3.32.
  4. Plumb `FLUTTER_TARGET_PLATFORM=windows-arm64` through `build.py` → `flutter build windows --target-platform=windows-arm64 --release`.
  5. **Preemptive** vcpkg.json fix: restrict `ffmpeg`, `mfx-dispatch`, and the FFmpeg `amf`/`nvcodec`/`qsv` features to `(x86 | x64) & windows` so they no longer activate on `arm64-windows-static`. (Even at the cost of `hwcodec`/`vram` on ARM64 — we're not building those features on this branch anyway, and the upstream port hard-blocks `nvcodec` on `arm64 & windows`.)
- Commit: pending

### Attempt 2 — Flutter 3.32.8 (x64 SDK) + vcpkg.json platform gates

- Date: 2026-05-27
- Run: https://github.com/johanclawson/rustdesk/actions/runs/26494409027
- Result: ❌ failed during `cargo build` of `magnum-opus`.
- What worked:
  - ✅ Flutter 3.32.8 (x64) installed, `flutter precache --windows` succeeded.
  - ✅ Rust toolchain reported as installed (`Install Rust toolchain (aarch64-pc-windows-msvc)`).
  - ✅ vcpkg installed all `arm64-windows-static` ports (mfx-dispatch/ffmpeg excluded by the manifest gates we added — verified by no `mfx-dispatch` build).
- Error:
  ```
  thread 'main' panicked at bindgen-0.59.2/src/lib.rs:2144:31:
    Unable to find libclang: "couldn't find any valid shared libraries matching:
    ['clang.dll', 'libclang.dll'], set the `LIBCLANG_PATH` environment variable
    to a path where one of these files can be found (invalid:
      [(C:\Program Files\LLVM\bin\libclang.dll: invalid DLL (x86-64)), ...])"
  ```
  And just above it, in the same `magnum-opus` build script stdout:
  ```
  cargo:info=x64-windows-static
  cargo:rustc-link-search=C:\vcpkg\installed\x64-windows-static\lib
  ```
- Diagnosis: **Two correlated bugs, both caused by x64 emulation hiding the real host arch.**
  1. `KyleMayes/install-llvm-action@v1` installed an **x86-64 libclang.dll** at `C:\Program Files\LLVM\bin`. The runner is ARM64 but the action's Node.js runs under x64 emulation, so its `RUNNER_ARCH` / `process.arch` detection picked x64. LLVM 15.0.6 publishes a `LLVM-15.0.6-woa64.exe` we should use directly.
  2. `dtolnay/rust-toolchain@v1` installed rustup with **x86_64-pc-windows-msvc as the default host triplet** (same reason — emulated x64 Node). Then `cargo build` without `--target` produces x86_64 binaries; `magnum-opus/build.rs` picks `x64-windows-static` from vcpkg; and even if libclang were fine, the resulting `librustdesk.dll` would be x64, not ARM64.

### Attempt 3 — install LLVM + rustup natively as ARM64

- Plan:
  1. Replace `install-llvm-action` with a manual `pwsh` step that downloads `LLVM-15.0.6-woa64.exe` directly and silent-installs it, then exports `LIBCLANG_PATH` so bindgen finds the ARM64 dll.
  2. Replace `dtolnay/rust-toolchain` with a manual `rustup-init.exe` from `static.rust-lang.org/rustup/dist/aarch64-pc-windows-msvc/`, passing `--default-host aarch64-pc-windows-msvc`. Now plain `cargo build` natively targets ARM64.
  3. Add a one-shot "Diagnose runner environment" step so future debugging has the raw `RUNNER_ARCH` / `RuntimeInformation` answers in the log.
- Commit: pending
