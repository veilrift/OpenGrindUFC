# OpenGrindUFC for iOS

### Open Grind for iOS (Unofficial Debug Build)

An unofficial iOS Client of [Open Grind](https://git.opengrind.org/open-grind/open-grind), for those who would like to try it out, created to make Open Grind usable on iPhone while waiting for an official iOS release from the original project.

**Download**: [IPA](https://git.opengrind.org/veilrift/OpenGrindUFC/releases)

---

> **Disclaimer:** OpenGrindUFC is **not an official Open Grind release**, and it is not affiliated with or endorsed by the original developers, I do not own Open Grind or claim authorship of the original project. OpenGrindUFC is an unofficial iOS Client, intended to make Open Grind available on iOS temporarily, until an official version is released by the original developers. All original code, trademarks, artwork, and other materials remain subject to their respective owners and licenses.

**Security:** While this build is not reproducable for iOS, it contanins the Original Source code with few patches without any critical modifications, feel free to clone this repository and inspect the code at any times. There won't be any modifications that do not belong to the Original Source code for security reasons.

**Important Notice:** Please refrain from reporting bugs and issues to the official project, feel free to open any Issues you have found to this Project.

**Credits**: CodexRunner (Regrind) for [BUILDING.MD](https://git.opengrind.org/codexrunner/regrind/src/branch/main/BUILDING.md)

## Screenshots

![App screenshots 3x2](https://git.opengrind.org/open-grind/open-grind/media/branch/main/contrib/app-screenshots-3x2.webp)

---

## Table of Contents

- [Patches & Fixes](#how-its-different-and-whats-fixed)
- [Requirements](#requirements)
- [Packages](#packages)
- [Building for iOS](#building-for-ios)
- [Installing on an iPhone](#installing-on-an-iphone)
- [Troubleshooting](#troubleshooting)
- [Original Project and Credits](#original-project-and-credits)
- [License](#license)

## Patches & Fixes

This section documents the changes made during the iOS port and debugging process.

### 1. Fixed the iOS launch crash

**Status: Fixed verified on a physical device.**

The application initially built and installed but crashed immediately when opened. Investigation pointed toward the iOS scene lifecycle and the way the `tao` Rust crate handled a scene configuration object.

The relevant function was:

```text
src-tauri/.patched/tao/src/platform_impl/ios/view.rs
```

Inside `configuration_for_connecting_scene_session`, the original code used:

```rust
Retained::as_ptr(&config) as _
```

This was changed to:

```rust
Retained::autorelease_ptr(config) as _
```

The change addresses the Objective-C object ownership and lifetime handoff at the scene configuration callback boundary.

**Result:** After the local dependency patch was selected and the app was rebuilt, OpenGrindUFC successfully opened on the iPhone.

### 2. Patched the local `tao` dependency

**Status: Fixed Cargo dependency resolution verified.**

Changing dependency source code alone does not guarantee that Cargo will compile the modified version. Initially, Cargo resolved `tao` from the crates.io registry rather than the local patched source.

A local patch was added in `src-tauri/Cargo.toml`:

```toml
[patch.crates-io]
tao = { path = ".patched/tao" }
```

The selected dependency was verified to resolve to:

```text
tao v0.35.3
src-tauri/.patched/tao
```

This ensures the modified iOS implementation is used during compilation.

**Maintenance note:** The local patch must remain available and committed. Dependency upgrades should be checked for an equivalent upstream fix before the patch is removed.

### 3. Added and checked the iOS scene manifest

**Status: Updated and included in the tested build.**

The generated iOS `Info.plist` initially lacked a `UIApplicationSceneManifest` configuration.

The manifest was updated to include an application scene configuration using:

- `UISceneConfigurationName`: `Default Configuration`
- `UISceneDelegateClassName`: `TaoSceneDelegate`
- `UIApplicationSupportsMultipleScenes`: `true`

The resulting IPA was inspected to confirm that the scene manifest was present.

Because generated project files may be overwritten during regeneration, this configuration should be verified in future builds and made reproducible through the project's configuration or generation process.

### 5. Fixed platform specific Rust compilation issues

**Status: iOS targeted build passed.**

Several changes were made to make platform-specific code compile correctly for iOS.

Updated SwiftRS from 1.0.7 to 1.0.9.

**Push backend**

Added an iOS fallback implementation:

```text
src-tauri/src/api/push/ios.rs
```

Updated conditional compilation in the push module so that iOS uses the iOS backend and desktop platforms use their desktop backend, with Android handled separately.

**Desktop only updater functionality**

Gated taskbar-related code in:

```text
src-tauri/src/api/update/install/desktop/mod.rs
```

using desktop-specific conditional compilation:

```rust
#[cfg(desktop)]
```

These changes prevent desktop only functionality from being compiled into incompatible platform targets.

### 6. Identified remaining compiler warnings

The build emitted warnings relating to unused methods, unnecessary `unsafe` blocks, unused variables, and references to mutable static variables.

These warnings did not prevent the application from building and launching.

They should be reviewed separately, particularly the mutable static warnings.

### Summary of Changes

| Area          | Change                                              | Result                                |
| ------------- | --------------------------------------------------- | ------------------------------------- | ---------- |
| iOS lifecycle | Corrected `tao` scene configuration pointer handoff | Launch crash resolved in tested build |
| Cargo         | Configured local `tao` patch                        | Patched code selected                 |
| iOS manifest  | Added scene configuration                           | Included in tested IPA                | in package |
| Push backend  | Added iOS fallback and platform selection           | iOS compilation supported             |
| Updater code  | Gated desktop only taskbar functionality            | Platform compatibility improved       |

---

## Requirements

Building the iOS adaptation requires a compatible macOS development environment.

- macOS
- Xcode and its iOS SDK, command-line tools
- Rust toolchain and required Apple compilation targets
- Bun, CocoaPods, Swiftly, Rustus, Homebrew & LLVM
- The Tauri CLI and iOS build prerequisites
- An iPhone and appropriate signing configuration for physical device testing

## Packages

[Homebrew](https://brew.sh/) - [Bun](https://bun.com/docs/installation) - [Rustup](https://rustup.rs/)

Install with Homebrew:

```
brew install npm
brew install cocoapods
brew install tauri
brew install LLVM
brew install cmake
```

The exact versions should match the project's pinned toolchain and dependency requirements.

## Building for iOS

### iOS build configuration

Everything needed to reproduce an iOS build is committed:

| Setting                | Value                      | Where it's pinned                                                             |
| ---------------------- | -------------------------- | ----------------------------------------------------------------------------- | --- |
| Apple development team | `APPLE_DEVELOPMENT_TEAM`   | your environment, see below (config holds a placeholder)                      |
| Bundle identifier      | `com.opengrind.ufc`        | `src-tauri/tauri.conf.json` &rarr; `identifier` (and `gen/apple/project.yml`) |     |
| iOS usage descriptions | location permission string | `src-tauri/Info.ios.plist`                                                    |

### 1. Obtain the source

Clone this Repository

### 2. Install dependencies

From the project root, install the JavaScript dependencies using the package manager and lockfile required by the checkout.

For a Bun-based project, this typically involves:

```bash
bun install
```

Ensure the Rust toolchain, iOS target, Xcode, and Tauri iOS prerequisites are configured before building.

### 3. Check the Rust iOS target

From the `src-tauri` directory:

```bash
cd src-tauri
cargo check --target aarch64-apple-ios --no-default-features
```

A successful check verifies that the Rust code compiles for the selected iOS target. It does not replace a complete IPA build or a device launch test.

### 4. Verify the local `tao` patch

Confirm that the local dependency exists:

```bash
cargo tree --target aarch64-apple-ios -i tao
```

The selected `tao` crate should resolve to the local `.patched/tao` directory. If it resolves to the registry copy, review `Cargo.toml` and `Cargo.lock` before proceeding.

### 5. Build the iOS application

Obtain your TeamID:
**Keychain Access > login > my certificates, double click at Apple Development and replace XXXXXXXXXX with Organization Unit Value**a

Export your TeamID to terminal:

```
export APPLE_DEVELOPMENT_TEAM=XXXXXXXXXX
```

From the repository root:

```bash
bun tauri ios build
```

The output location depends on the build configuration. In the tested setup, the IPA was generated at:

```text
src-tauri/gen/apple/build/arm64/OpenGrindUFC.ipa
```

### 6. Verify the package

Before installation, verify the bundle identifier and inspect the packaged scene manifest.

The expected bundle identifier is:

`YOUR SPECIFIED BUNDLE ID`

Ensure that the IPA was produced by the intended checkout and includes the local fixes.

## Installing on an iPhone

Obtain your UDID:

```
xcrun devicectl device install app \
  --device (YOUR-UDID) \
  "~/open-grind/src-tauri/gen/apple/build/arm64/OpenGrindUFC.ipa"
```

## Troubleshooting

### The app crashes immediately on launch

1. Confirm that the latest build includes the local `tao` patch.
2. Check that Cargo resolves `tao` to `.patched/tao`.
3. Verify the scene manifest and bundle identifier in the packaged IPA.
4. Rebuild the application and install the newly generated package.
5. If the crash persists, inspect the newest device crash report in Xcode's Devices and Simulators window.

### Cargo uses the unpatched dependency

Run:

```bash
cd src-tauri
cargo tree --target aarch64-apple-ios -i tao
```

Check that the output shows the local patched path. Confirm the `[patch.crates-io]` entry and ensure the local crate version satisfies the dependency requirements.

### The build emits warnings

Warnings about unused code or unnecessary `unsafe` blocks may not prevent compilation. However, warnings about mutable static references should be reviewed rather than ignored. If the build fails, identify the first actual `error:` message and inspect the final compiler output.

### The app installs but still does not open

Confirm that the correct, newly built IPA was installed and that it uses `com.opengrind`. Retrieve the latest device crash report and investigate the actual exception rather than assuming the previous issue has returned.

---

## Original Project and Credits

OpenGrindUFC builds on the work of the original Open Grind project.

**Original project:** [Open Grind](https://git.opengrind.org/open-grind/open-grind)

**Original developer:** Viktor Shchelochkov

**Additional credits:** The Open Grind maintainers and contributors whose code, fixes, documentation, and community work make the original project possible.

Open Grind is described by its developers as an unofficial, cross platform Grindr client. The original project includes its own source code, build instructions, license information, and contribution guidelines.

Please visit the original repository for the authoritative source, project updates, and licensing details:

- [Official Open Grind repository](https://git.opengrind.org/open-grind/open-grind)
- [Original project's README](https://git.opengrind.org/open-grind/open-grind/src/branch/main/README.md)
- [Original project's build instructions](https://git.opengrind.org/open-grind/open-grind/src/branch/main/BUILDING.md)
- [Original project's license](https://git.opengrind.org/open-grind/open-grind/src/branch/main/LICENSE)

Thank you to Viktor Shchelochkov and all Open Grind contributors for creating and maintaining the underlying project.

## License

Original project's [LICENSE](https://git.opengrind.org/open-grind/open-grind/src/branch/main/LICENSE) and [ARTWORK.md](https://git.opengrind.org/open-grind/open-grind/src/branch/main/ARTWORK.md) for authoritative terms.
