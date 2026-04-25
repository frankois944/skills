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

Always load [`references/exporting.md`](references/exporting.md) and [`references/dependencies.md`](references/dependencies.md) for Scenario B. For **every product** the user adds, perform two checks before writing the Gradle config: (1) look up the package's minimum deployment target in its `Package.swift` and update `minIos`/`minMacos`/`minTvos`/`minWatchos` accordingly; (2) determine ObjC compatibility via the modulemap detection steps and set `exportToKotlin = true` or `false`. Never skip either check.

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

6. **For every dependency added, check the package's minimum deployment target and raise `minIos`/`minMacos`/`minTvos`/`minWatchos` in `swiftPackageConfig` if needed.** Look at the package's `Package.swift` `.platforms` declaration. Setting a lower version than the package requires causes a build error. The defaults (`minIos = "12.0"`, `minMacos = "10.13"`, `minTvos = "12.0"`, `minWatchos = "4.0"`) are often too low for modern packages.

7. **Always set `exportToKotlin` explicitly — never omit it.** For each product, run the detection steps in [`references/exporting.md`](references/exporting.md) § "Detecting ObjC Compatibility via the Modulemap" and set `exportToKotlin = true` only when the check confirms ObjC compatibility on the **direct target** backing the product. Transitive dependencies do not count — cinterop only sees the direct target's interface. If the check cannot be performed (inaccessible source, closed-source binary), set `exportToKotlin = false`. When in doubt, `false` is always the safe default.
