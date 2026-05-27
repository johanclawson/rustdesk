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
- Run: https://github.com/johanclawson/rustdesk/actions/runs/26495736490
- Result: ❌ failed during link of `rustdesk` build script with 132 unresolved `libsodium` symbols.
- What worked:
  - ✅ Native woa64 LLVM installed (clang.exe reports ARM64).
  - ✅ rustup installed with `aarch64-pc-windows-msvc` as the *default host*.
  - ✅ vcpkg-rs in build scripts now reports `cargo:info=arm64-windows-static` — `magnum-opus` linked cleanly.
- Error:
  ```
  liblibsodium_sys-...rlib : warning LNK4272: library machine type 'x64' conflicts with target machine type 'ARM64'
  unresolved external symbol crypto_box_easy ... (× 132)
  ```
- Diagnosis: `libsodium-sys = "0.2.7"` (transitive via `sodiumoxide`) bundles a "download prebuilt libsodium.lib" path on MSVC. The Jedisct1 distribution only ships x86 + x64 prebuilts, so on aarch64 it downloads the x64 .lib and tries to link it anyway. The `LNK4272` warning is the smoking gun.
- libsodium-sys's `build.rs` (verified upstream) honours `SODIUM_LIB_DIR` and will then link `libsodium.lib` from that directory instead of downloading anything.

### Attempt 4 — vcpkg arm64 libsodium + `SODIUM_LIB_DIR`

- Plan:
  1. Add `libsodium` to `vcpkg.json` gated to `arm64 & windows & static` (does not touch the x64 build path).
  2. Export `SODIUM_LIB_DIR=C:\vcpkg\installed\arm64-windows-static\lib` for the build step. libsodium-sys will then pick up the ARM64 `libsodium.lib` vcpkg just produced.
- Run: https://github.com/johanclawson/rustdesk/actions/runs/26496990390
- Result: ❌ failed during `magnum-opus` build script with `opus_multistream.h not found`.
- Diagnosis turn-up: `magnum-opus/build.rs` (and `libs/scrap/build.rs`, and `rustdesk-org/hwcodec/build.rs`) all contain the same bug:
  ```rust
  } else if target_os == "windows" {
      "x64-windows-static".to_owned()   // ← hard-coded, ignores target_arch
  }
  ```
  i.e. they detect `aarch64` correctly but then throw it away for any Windows target. Result: every vcpkg lookup goes to `C:\vcpkg\installed\x64-windows-static\…` and finds nothing (we only installed `arm64-windows-static`). The libsodium fix from Attempt 3 *did* work — the link error never even reproduced because `magnum-opus` now fails before linking.

### Proactive audit (before attempt 5)

Survey of every build.rs that participates in the active build:

| location | bug | active here? |
|---|---|---|
| `libs/scrap/build.rs` | hard-codes `x64-windows-static` | yes |
| `rustdesk-org/magnum-opus@5cd2bf98/build.rs` | hard-codes `x64-windows-static` | yes |
| `rustdesk-org/hwcodec@398e5a89/build.rs` | hard-codes `x64-windows-static` | no — feature `hwcodec` is off |
| `rustdesk-org/cpal`, `kcp-sys`, `impersonate-system`, `rdev`, `arboard`, `clipboard-master`, `The-Fat-Controller`, `nokhwa-bindings-windows` | none touch vcpkg triplets | yes |
| `libs/enigo`, `libs/clipboard`, `libs/virtual_display/dylib` build.rs | only `cfg(target_os)`, no triplets | yes |
| `libs/hbb_common/build.rs` | only `protobuf_codegen` | yes |
| `libsodium-sys 0.2.7` | downloads x64 prebuilt | yes — already mitigated by `SODIUM_LIB_DIR` |

### Attempt 5 — fix the triplet hard-coding in our copy of scrap, alias for the rest

- Plan:
  1. Patch `libs/scrap/build.rs` line 49–50: replace the literal `"x64-windows-static"` with `format!("{}-windows-static", target_arch)`. This is upstream-PR-worthy and avoids the workaround for at least one crate.
  2. For the external git deps we can't trivially patch from this branch (`magnum-opus`, plus `hwcodec` if/when we re-enable it), add a workflow step that creates an NTFS directory junction `C:\vcpkg\installed\x64-windows-static → C:\vcpkg\installed\arm64-windows-static`. Whatever the hard-coded crates point at is now backed by the ARM64 libraries — including a libsodium.lib at the x64 path, which means our `SODIUM_LIB_DIR` env from Attempt 4 keeps working.
- Follow-ups for the eventual upstream PR (separate from getting *this* build green):
  - Send PRs to `rustdesk-org/magnum-opus`, `rustdesk-org/hwcodec`, and any other rustdesk fork with the same `else if target_os == "windows" { "x64-windows-static" }` pattern.
- Run: https://github.com/johanclawson/rustdesk/actions/runs/26498557896
- Result: ❌ failed at `flutter build windows --target-platform=windows-arm64 --release`. Cargo build of `librustdesk.dll` for `aarch64-pc-windows-msvc` SUCCEEDED ("Finished release [optimized] target(s) in 8m 52s"). 🎉
- New error:
  ```
  Could not find an option named "--target-platform".
  ```
- Diagnosis: `flutter build windows` does NOT accept `--target-platform` in any Flutter version (3.32, 3.44 stable, master). Reading `packages/flutter_tools/lib/src/commands/build_windows.dart`:
  ```dart
  final defaultTargetPlatform =
      (_operatingSystemUtils.hostPlatform == HostPlatform.windows_arm64)
          ? 'windows-arm64' : 'windows-x64';
  ```
  And `lib/src/base/os.dart`:
  ```dart
  final abi = Abi.current();
  _hostPlatform = (abi == Abi.windowsArm64) ? HostPlatform.windows_arm64
                                            : HostPlatform.windows_x64;
  ```
  `Abi.current()` is a **compile-time** property of the Dart VM. So the x64 Flutter SDK (which subosito/flutter-action installs) ALWAYS reports x64 and ALWAYS builds windows-x64, irrespective of host emulation, env vars, or flags. **The only way to produce a windows-arm64 build is with an ARM64 Dart VM in the Flutter SDK.**

### Discovery — Flutter ARM64 artifacts ship for 3.44.0+

A walk of `https://storage.googleapis.com/storage/v1/b/flutter_infra_release/o?prefix=flutter/<engine>/`:

| Flutter ver | engine hash | `dart-sdk-windows-arm64.zip` | `windows-arm64-*` engine zips |
|---|---|---|---|
| 3.32.8 | ef0cd00091… | ❌ | ❌ (only `windows-x64-*`) |
| **3.44.0** | 4c525dac5e… | ✅ | ✅ (`debug` / `profile` / `release`) |

So Flutter 3.44.0 is the first stable whose engine actually publishes the ARM64 Windows desktop artifacts.

And `flutter/bin/internal/update_dart_sdk.ps1` does this:
```powershell
if ($env:PROCESSOR_ARCHITECTURE -eq "ARM64") {
    $dartSdkArm64Url = "$dartSdkBaseUrl/flutter_infra_release/flutter/$engineVersion/$dartZipNameArm64"
    ...
}
```
i.e. when run from a real ARM64 process it auto-downloads the arm64 dart-sdk. `subosito/flutter-action` ships a *pre-extracted* x64 zip whose bundled dart-sdk matches the version stamp, so update_dart_sdk.ps1 never runs and we're stuck on x64. But if we `git clone` Flutter ourselves (no bundled dart-sdk), the first `flutter --version` triggers update_dart_sdk.ps1 — and pwsh on the windows-11-arm runner runs as native ARM64 (`PROCESSOR_ARCHITECTURE=ARM64`), so we get the arm64 sdk.

### Attempt 6 — git-clone Flutter 3.44.0 so bootstrap pulls ARM64 dart-sdk

- Plan:
  1. Drop `subosito/flutter-action`. Replace with a pwsh step that does `git clone --depth=1 --branch 3.44.0 https://github.com/flutter/flutter.git C:\flutter` and prepends `C:\flutter\bin` to PATH.
  2. Run `flutter --version` once (triggers `update_dart_sdk.ps1`, downloads `dart-sdk-windows-arm64.zip`).
  3. `flutter precache --windows --no-android --no-ios --no-linux --no-macos --no-web --no-fuchsia` (the windows-arm64-flutter.zip engine artifacts now exist for 3.44.0).
  4. Bump `FLUTTER_VERSION` env to `3.44.0`. Drop the `FLUTTER_TARGET_PLATFORM` env from the build step — host detection is now arm64 so `flutter build windows --release` does the right thing on its own.
- Risk: rustdesk's Flutter side was last patched against 3.24.x. Jumping straight to 3.44.0 might surface pubspec/widget/API mismatches. Pubspec only constrains `sdk: '^3.1.0'` (passes Dart 3.12), so the most likely failure is a deprecated widget API. We'll see in the next run.
- Run: https://github.com/johanclawson/rustdesk/actions/runs/26500599054
- Result: ❌ failed at `flutter build windows --release`. **But cargo native ARM64 build succeeded** AND **Flutter is now producing `flutter\build\windows\arm64\...`** — the arch-detection plumbing works. 🎉
- Errors are all Flutter 3.44 / Dart 3.12 API mismatches in the rustdesk Flutter code or its package pins:
  ```
  extended_text-14.0.0/lib/src/official/rendering/paragraph.dart(1124): The non-abstract class
    '_SelectableFragment' is missing implementations for these members…
  extended_text-14.0.0/lib/src/extended/selection_mixin.dart(89): The non-abstract class
    '_ExtendedSelectableFragment' is missing implementations for these members…
  lib/common.dart(384): The argument type 'DialogTheme' can't be assigned to
    the parameter type 'DialogThemeData?'.
  lib/common.dart(415): The argument type 'TabBarTheme' can't be assigned to
    the parameter type 'TabBarThemeData?'.
  google_fonts-6.2.1/lib/src/google_fonts_variant.dart(152): Constant evaluation error
  ```

### Dead-end check — earliest Flutter with ARM64 artifacts

I walked engine hashes for every Flutter 3.27…3.44 stable + 3.42/3.43 betas; only **3.44.0** publishes `dart-sdk-windows-arm64.zip` and the `windows-arm64-*` engine artifacts. We can't downgrade to a Flutter that's gentler on rustdesk's Dart code.

### Attempt 7 — forward-port rustdesk Flutter code to 3.44

- Plan (surgical, minimal-diff):
  1. `pubspec.yaml`: bump `extended_text 14.0.0` → `^15.0.2` (15.x supports Dart ≥3.7, the only version family that compiles on Flutter 3.44). Bump `google_fonts ^6.2.1` → `^8.0.0` (8.x is the current line, fixes the const-eval bug). Both packages' rustdesk usage is trivial (`ExtendedText(..., maxLines: 1)`, `GoogleFonts.robotoMono().fontFamily`), so no API-side changes expected.
  2. `flutter/lib/common.dart`: rename the four `DialogTheme(…)` / `TabBarTheme(…)` constructor calls to `DialogThemeData(…)` / `TabBarThemeData(…)`. (Flutter 3.44 renamed the classes — the field names `dialogTheme:` / `tabBarTheme:` are unchanged.)
  3. `flutter/lib/desktop/widgets/tabbar_widget.dart`: drop `hide TabBarTheme` from the material.dart import (the symbol no longer exists; nothing else in this file references it).
  4. Add `flutter/**` and `libs/**` to the workflow's `paths:` trigger so Dart/Rust source changes re-run CI without a workflow edit.
- Open question: there may be more 3.24→3.44 API drift past these three points. We'll know after the run.
- Run: https://github.com/johanclawson/rustdesk/actions/runs/26502756584
- Result: ❌ failed *earlier* than before — the **bridge** job (not the ARM64 job) now fails. The bumped `extended_text ^15.0.2` requires Dart 3.7+, but `bridge.yml` runs on Flutter 3.22.3 (Dart 3.4.4):
  ```
  The current Dart SDK version is 3.4.4.
  Because flutter_hbb depends on extended_text >=14.0.0 which requires SDK version >=3.5.0 <4.0.0,
  version solving failed.
  ```
  Bridge already had a `sed -i 's/extended_text: 14.0.0/extended_text: 13.0.0/g' pubspec.yaml` workaround for its own old Dart, but my new pubspec line `extended_text: ^15.0.2` doesn't match that pattern, and the same problem now also applies to `google_fonts: ^8.0.0`.

### Attempt 8 — extend the bridge sed to back-port our pubspec bumps

- Plan:
  - In `bridge.yml`, broaden the sed to also rewrite `extended_text: ^15.x` → `13.0.0` and `google_fonts: ^8.x` → `^6.2.1` just for the bridge run. The actual ARM64 build still sees the real `^15.0.2` / `^8.0.0`.
- This change to bridge.yml is bridge-job-local — the generated bridge files only depend on `src/flutter_ffi.rs`, not on which pubspec versions resolve.
- Commit: pending
