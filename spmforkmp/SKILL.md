---
name: spmforkmp
description: >
  Use this skill whenever working with the spmForKmp Gradle plugin (id: `io.github.frankois944.spmForKmp`) — a modern replacement for the deprecated Kotlin CocoaPods plugin for integrating Swift Package Manager (SPM) dependencies into Kotlin Multiplatform (KMP) projects targeting Apple platforms.

  Trigger on any of these: configuring the spmForKmp plugin, adding Swift Package dependencies to a KMP project, writing Swift bridge code (`@objcMembers`) for Kotlin Multiplatform, migrating from `org.jetbrains.kotlin.native.cocoapods`, exporting Objective-C compatible packages to Kotlin (`exportToKotlin`), troubleshooting `Undefined symbol` or cinterop errors in KMP Apple targets, questions about `swiftPackageConfig {}`, Swift↔Kotlin interop, or anyone saying "spm4kmp", "spmForKmp", or "swift package" in a KMP/Kotlin Multiplatform context. Use this skill even if the user doesn't name the plugin explicitly — if they are trying to use a Swift library from Kotlin on an Apple platform, this skill applies.
---

## What spmForKmp Does

`spmForKmp` compiles Swift code and SPM packages into a `cinterop`-based bridge that Kotlin can call on Apple targets. It handles:

- **Bridge**: Your Swift files in `src/swift/[bridgeName]/` are compiled and exposed to Kotlin via cinterop
- **SPM Dependencies**: External Swift packages fetched and linked into the bridge
- **Export to Kotlin**: ObjC-compatible packages exposed directly without a bridge wrapper
- **Migration**: Drop-in replacement for the `cocoapods {}` DSL

## How to Help the User

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

8. **Binary XCFramework dependencies require `exportedPackageSettings` and must be wired into Xcode manually.** Detection: look at the dependency's own `Package.swift` (or the `scratch/artifacts/<package>/<Product>/<Product>.xcframework/` path that appears in the plugin's `.def` files after a build) — the presence of `.binaryTarget(name: "X", ...)` or a `.xcframework` artifact means X is binary. Source packages (ObjC or Swift) are compiled into `libnativeBridge.a` and linked automatically; binary targets are not — their symbols will be undefined at Xcode link time.

   **Apply at config time, not at troubleshooting time.** Add `exportedPackageSettings { includeProduct = listOf("ProductName") }` to the `swiftPackageConfig` block in the same edit that adds the dependency. After the Gradle build, a local Swift package is generated (the plugin prints its path, typically `<module>/exportedNativeBridge`). Wire it into `project.pbxproj` — the five edits required:

   1. New `XCLocalSwiftPackageReference` section pointing at the local package's relative path.
   2. New `XCSwiftPackageProductDependency` section with a **mandatory** `package = <local-ref-UUID>` field. Omitting `package` orphans the product and causes `Missing package product '...'`.
   3. New `PBXBuildFile` entry referencing the product dependency via `productRef`.
   4. Add the local-ref UUID to `packageReferences` inside `PBXProject` — **never** `localPackages` (unrecognised by xcodebuild).
   5. Add the product-dep UUID to the target's `packageProductDependencies` AND the build-file UUID to the target's `PBXFrameworksBuildPhase.files` list.

   Always run `xcodebuild -resolvePackageDependencies -project …` **before** `xcodebuild … build`. Building without resolving first fails with `Missing package product` even when the pbxproj is correct.

9. **A migration is not complete until the app builds in Xcode AND launches on a simulator.** Gradle success only proves the bridge compiles. The Xcode build proves linking and embedding work; the simulator launch proves the framework loads at runtime and there are no missing symbols. Always finish with the verification flow in [`references/migration.md`](references/migration.md) § "Final Verification" — `xcodebuild build` against the project (never the deleted `.xcworkspace`) on the first available iPhone simulator, then `simctl install` + `simctl launch`. If launch fails, do not report success.

   The bridge folder also needs at least one `.swift` file even when `exportToKotlin = true` for every product. Swift PM rejects an "empty" target with `target '<bridgeName>' referenced in product '<bridgeName>' is empty`, so a `.gitkeep` is not enough — drop in a one-line `Bridge.swift` containing `import Foundation`.
