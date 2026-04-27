---
name: spmforkmp
license: MIT
compatibility: Works with any Agent Skills compatible environment (Junie, Claude Code, Cursor, etc.).
description: >
  Use this skill whenever working with the spmForKmp Gradle plugin (id: `io.github.frankois944.spmForKmp`) — a modern replacement for the deprecated Kotlin CocoaPods plugin for integrating Swift Package Manager (SPM) dependencies into Kotlin Multiplatform (KMP) projects targeting Apple platforms.

  Trigger on any of these: configuring the spmForKmp plugin, adding Swift Package dependencies to a KMP project, writing Swift bridge code (`@objcMembers`) for Kotlin Multiplatform, migrating from `org.jetbrains.kotlin.native.cocoapods`, exporting Objective-C compatible packages to Kotlin (`exportToKotlin`), troubleshooting `Undefined symbol` or cinterop errors in KMP Apple targets, questions about `swiftPackageConfig {}`, Swift↔Kotlin interop, or anyone saying "spm4kmp", "spmForKmp", or "swift package" in a KMP/Kotlin Multiplatform context. Use this skill even if the user doesn't name the plugin explicitly — if they are trying to use a Swift library from Kotlin on an Apple platform, this skill applies.
---

## Definition of Done — Read This First

**A migration or new integration is NOT complete until `xcrun simctl launch` returns a PID for the app on a booted iPhone simulator.** This is a hard, blocking acceptance criterion — not a "nice to have" verification step. Gradle BUILD SUCCESSFUL is necessary but never sufficient.

Concretely, before reporting success you must have observed, in this order:

1. `./gradlew :<app-module>:linkDebugFrameworkIosSimulatorArm64` succeeds.
2. `xcodebuild … build` against the iOS app's `.xcodeproj` (never a deleted `.xcworkspace`) succeeds.
3. `xcrun simctl install <UDID> <path>.app` succeeds.
4. `xcrun simctl launch <UDID> <bundle-id>` returns a non-zero PID without an immediate crash.

The full command sequence is in [`references/migration.md`](references/migration.md) § "Final Verification — Build and Launch on a Simulator". If any step fails, treat the migration as in-progress and continue debugging — do not declare success, do not summarize as "complete," do not stop until launch succeeds. If the user's environment genuinely cannot boot a simulator (e.g. CI without a Mac runner), say so explicitly and surface the unverified status — never silently elide the step.

## What spmForKmp Does

`spmForKmp` compiles Swift code and SPM packages into a `cinterop`-based bridge that Kotlin can call on Apple targets. It handles:

- **Bridge**: Your Swift files in `src/swift/[bridgeName]/` are compiled and exposed to Kotlin via cinterop
- **SPM Dependencies**: External Swift packages fetched and linked into the bridge
- **Export to Kotlin**: ObjC-compatible packages exposed directly without a bridge wrapper
- **Migration**: Drop-in replacement for the `cocoapods {}` DSL

## How to Help the User

**Before writing any Gradle snippet**, fetch [plugins.gradle.org/plugin/io.github.frankois944.spmForKmp](https://plugins.gradle.org/plugin/io.github.frankois944.spmForKmp) to confirm the current plugin version and use that value in every code example. Never hard-code a version without verifying it first.

Identify the scenario, load the relevant reference file(s), then produce complete Gradle config + Swift code + Kotlin usage. Always produce the full picture — don't give only a dependency snippet without showing how the bridge and Kotlin usage fit together.

### The Three Core Scenarios

**Scenario A — Basic integration (no external SPM package)**
The user only needs Apple SDK access (UIKit, Foundation, CoreLocation…) from Kotlin.
→ Load [`references/setup.md`](references/setup.md) + [`references/bridge.md`](references/bridge.md)

**Scenario B — Integration with an external SPM package**
The user wants to add a third-party Swift package. Pick the dependency type that fits:

| Source | Type | Load |
|---|---|---|
| Published GitHub/GitLab release tag | Remote — version | [`references/dependencies.md`](references/dependencies.md) § Remote Version |
| Git branch (main, develop…) | Remote — branch | [`references/dependencies.md`](references/dependencies.md) § Remote Branch |
| Specific git commit | Remote — commit | [`references/dependencies.md`](references/dependencies.md) § Remote Commit |
| Local Swift package on disk | Local — source | [`references/dependencies.md`](references/dependencies.md) § Local Package |
| Local `.xcframework` | Local — binary | [`references/dependencies.md`](references/dependencies.md) § Local Binary |
| Remote `.xcframework` zip | Remote — binary | [`references/dependencies.md`](references/dependencies.md) § Remote Binary |

Always load [`references/exporting.md`](references/exporting.md) and [`references/dependencies.md`](references/dependencies.md) for Scenario B. For **every product** the user adds, perform three checks before writing the Gradle config — never skip any of them:

1. **Min OS** — look up the package's `.platforms` declaration in its `Package.swift` and raise `minIos`/`minMacos`/`minTvos`/`minWatchos` if the new requirement is higher than the current value.
2. **ObjC compatibility** — run the modulemap detection in [`references/exporting.md`](references/exporting.md) and set `exportToKotlin = true` or `false`.
3. **Target type (binary vs. source)** — open the dependency's `Package.swift` and check whether the target backing the product is `.binaryTarget(...)` (XCFramework, often used by closed-source vendor SDKs like GoogleMaps, Firebase Crashlytics, etc.). If yes, add `exportedPackageSettings { includeProduct = listOf("ProductName") }` to the same `swiftPackageConfig` block **and wire the auto-generated local Swift package into `project.pbxproj`**. See rule 8 for the pbxproj edits. Skipping this check at config time produces a green Gradle build that fails Xcode link with `Undefined symbol: _OBJC_CLASS_$_…` — caught only by rule 9's simulator launch, never by Gradle alone.

**Scenario C — Migrate from the KMP CocoaPods plugin**
→ Load [`references/migration.md`](references/migration.md). It contains the full before/after for Gradle, Xcode, and import statements.

### Other references

| Need | Load |
|---|---|
| Advanced interop patterns (enums, async/await, throws, struct wrapping) | [`references/interoperability.md`](references/interoperability.md) |
| Troubleshoot build/link/runtime error | [`references/troubleshooting.md`](references/troubleshooting.md) |

## Non-Negotiable Rules

Always apply these — they are the most common sources of user pain:

1. **`@objcMembers public class ... : NSObject` is required.** Any Swift class, method, or property that must be visible to Kotlin needs `@objcMembers` (or per-item `@objc`) and `public`. Pure Swift types (structs, enums, Swift-only protocols, actors, async/await) cannot cross the boundary.

2. **`import io.github.frankois944.spmForKmp.swiftPackageConfig` is required at the top of `build.gradle.kts`.** Without this import the function resolves to a Gradle built-in with an incompatible signature and the build fails with "receiver type mismatch". Call it as `target.swiftPackageConfig("bridgeName") { ... }` — the bridge name is a positional string, not `cinteropName =`.

3. **Use the same bridge name across all Apple targets.** Passing the same string to `swiftPackageConfig` on every target points them all to the same `src/swift/[bridgeName]/` folder. Without a shared name each target gets its own directory.

4. **`gradle.properties` must contain `kotlin.mpp.enableCInteropCommonization=true`.** Without it, the shared cinterop definitions will not resolve.

5. **Kotlin 2.x requires opting into `ExperimentalForeignApi` at cinterop call sites.** The cleanest way is one line per target in Gradle: `target.compilerOptions.optIn.add("kotlinx.cinterop.ExperimentalForeignApi")`. Alternatively annotate individual Kotlin files with `@OptIn(kotlinx.cinterop.ExperimentalForeignApi::class)`.

6. **Every time a dependency or product is added, re-check the minimum OS version and update the plugin config.** Read the new package's `.platforms` in its `Package.swift` and raise `minIos`/`minMacos`/`minTvos`/`minWatchos` if the new requirement exceeds the current value. The configured minimum must always reflect the highest requirement across all dependencies in the block — each new addition may raise the bar. Never skip this step: a mismatch causes a build error. If the package has no `.platforms` block, no change is needed.

7. **Always set `exportToKotlin` explicitly — never omit it.** For each product, run the detection steps in [`references/exporting.md`](references/exporting.md) § "Detecting ObjC Compatibility via the Modulemap" and set `exportToKotlin = true` only when the check confirms ObjC compatibility on the **direct target** backing the product. Transitive dependencies do not count — cinterop only sees the direct target's interface. If the check cannot be performed (inaccessible source, closed-source binary), set `exportToKotlin = false`. When in doubt, `false` is always the safe default.

8. **Some dependencies must be added to Xcode as a local package — the plugin tells you which ones.** After each Gradle build, the plugin evaluates every dependency using three criteria and generates a local Swift package for those that need Xcode-side linking:
   - **Always included**: C/ObjC modules (`isCLang`) are unconditionally added.
   - **Explicitly included**: any product listed in `exportedPackageSettings { includeProduct = listOf("ProductName") }`.
   - **Auto-detected**: the plugin scans compiled output dirs for static `.framework` files (reads `Info.plist`, calls `isDynamicLibrary`) and includes any static framework it finds. Detection is not 100% reliable — binary XCFrameworks in particular may be missed.

   When the local package is generated and `hideLocalPackageMessage` is false, the plugin prints a `logger.error` message (always visible, even without `--info`):
   ```
   Spm4Kmp: The following dependencies [ProductName] need to be added to your xcode project
   A local Swift package has been generated at /path/to/exportedNativeBridge
   Please add it to your xcode project as a local package dependency
   ```
   Add that generated package to your Xcode project as a **local package dependency** (File → Add Package Dependencies → Add Local… in Xcode). Once added, set `spmforkmp.hideLocalPackageMessage=true` in `gradle.properties` to silence the message.

   If a dependency is NOT auto-detected and causes `Undefined symbol` at Xcode link time, add it explicitly: `exportedPackageSettings { includeProduct = listOf("ProductName") }` in the same `swiftPackageConfig` block, then rebuild.

   Always run `xcodebuild -resolvePackageDependencies -project …` **before** `xcodebuild … build` after adding the local package. Building without resolving first fails with `Missing package product` even when the project is correctly configured.

   For manual `project.pbxproj` editing (CI or no Xcode UI), see `references/troubleshooting.md` § "Undefined symbol".

9. **A migration is not complete until `xcrun simctl launch` returns a PID.** This is the definition of done for every spmForKmp task — see the "Definition of Done" section at the top of this file. Gradle success only proves the bridge compiles; the Xcode build proves linking and embedding; the simulator launch proves the framework loads at runtime with no missing symbols. Run the full sequence in [`references/migration.md`](references/migration.md) § "Final Verification" — `xcodebuild build` against the `.xcodeproj` (never the deleted `.xcworkspace`) on the first available iPhone simulator, then `simctl install` + `simctl launch`. If any step fails or you cannot run the simulator, do **not** report success — keep debugging or surface the unverified status explicitly.

   The bridge folder also needs at least one `.swift` file even when `exportToKotlin = true` for every product. Swift PM rejects an "empty" target with `target '<bridgeName>' referenced in product '<bridgeName>' is empty`, so a `.gitkeep` is not enough — drop in a one-line `Bridge.swift` containing `import Foundation`.
