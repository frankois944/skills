# Setup & Dependencies Reference

## Requirements

| Tool | Minimum |
|---|---|
| macOS + Xcode | 16+ |
| Kotlin | 2.1.0+ |
| Gradle | 8.12+ |

## 1. Apply the Plugin

### With version catalog (`libs.versions.toml`)

```toml
[versions]
spmForKmp = "1.9.1"   # check Gradle Plugin Portal for latest

[plugins]
spmForKmp = { id = "io.github.frankois944.spmForKmp", version.ref = "spmForKmp" }
```

Root `build.gradle.kts`:
```kotlin
plugins {
    alias(libs.plugins.spmForKmp).apply(false)
}
```

Module `build.gradle.kts`:
```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

plugins {
    alias(libs.plugins.spmForKmp)
}
```

### Without version catalog

```kotlin
// Module build.gradle.kts
import io.github.frankois944.spmForKmp.swiftPackageConfig

plugins {
    id("io.github.frankois944.spmForKmp") version "1.9.1"
}
```

## 2. gradle.properties

```properties
kotlin.mpp.enableCInteropCommonization=true
```

## 3. `swiftPackageConfig` Block

### Multiple targets (recommended pattern)

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(
        iosArm64(),
        iosSimulatorArm64(),
        // add macosArm64(), tvosArm64(), etc. as needed
    ).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            // creates src/swift/nativeBridge/
            // all targets share the same bridge folder and config
        }
    }
}
```

### Single target

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    iosArm64 {
        swiftPackageConfig("iosArm64Bridge") {
            // creates src/swift/iosArm64Bridge/
        }
    }
}
```

### Multiple platform families (iOS + macOS)

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeIosBridge") {
            minIos = "16.0"
        }
    }
    macosArm64 {
        swiftPackageConfig("macosBridge") {
            // separate bridge for macOS
        }
    }
}
```

## 4. Key `swiftPackageConfig` Options

| Property | Default | Notes |
|---|---|---|
| `minIos` | `"12.0"` | Set `null` to omit from Package.swift |
| `minMacos` | `"10.13"` | |
| `minTvos` | `"12.0"` | |
| `minWatchos` | `"4.0"` | |
| `debug` | `false` | Enable during dev for debug symbols |
| `toolsVersion` | `"5.9"` | Swift tools version in Package.swift |
| `spmWorkingPath` | `{buildDir}/spmKmpPlugin/` | Move outside build dir to survive `./gradlew clean` |

### Performance tip: persist the SPM working directory

```kotlin
target.swiftPackageConfig("nativeBridge") {
    spmWorkingPath = "${projectDir}/SPM"  // not deleted by clean
}
```

Cache this path in CI/CD to avoid re-downloading packages on every run.

## 5. Kotlin 2.x: ExperimentalForeignApi opt-in

Kotlin 2.x requires opting into `ExperimentalForeignApi` for any cinterop call site. The cleanest approach is enabling it for all compilations (main and test) in Gradle — avoids annotating every Kotlin file:

```kotlin
listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
    target.compilations.all {
        compileTaskProvider.configure {
            compilerOptions.optIn.add("kotlinx.cinterop.ExperimentalForeignApi")
        }
    }
    target.swiftPackageConfig("nativeBridge") { ... }
}
```

Alternatively, annotate individual call sites:
```kotlin
@OptIn(kotlinx.cinterop.ExperimentalForeignApi::class)
actual fun platformInfo(): String = MyBridge().appVersion()
```

## 6. After Setup

The first sync creates `src/swift/[cinteropName]/StartYourBridgeHere.swift` — a template with examples. Delete it once you add real code, or set `spmforkmp.disableStartupFile=true` in `gradle.properties` to prevent it from being generated. If no file exists inside the bridge folder, the plugin generates an empty one during the build — no need to create a dummy file.

Keep that file (and don't add a `.gitkeep`) and the empty-target issue never appears. Only run into this when you've manually pre-created the folder structure (e.g. during a CocoaPods migration before the first Gradle sync).

---

## 7. Adding External Dependencies

Each section below is a complete, self-contained recipe. Always produce: Gradle config + Kotlin usage. Only produce Swift bridge code if the user explicitly asks for it.

For every dependency added, these three steps are **mandatory** before writing any Gradle config. Do not skip them.

**Step 1 — Update the minimum OS version (required for every new dependency or product)**
Each time a dependency or product is added, read its package-level `.platforms` declaration in `Package.swift` and update `minIos`, `minMacos`, `minTvos`, and `minWatchos` in `swiftPackageConfig` if the new requirement is higher than the current value. The configured minimum must always be the highest value required by any dependency in the block.

```swift
// New dependency requires iOS 16 — current config has minIos = "15.0"
// → raise to minIos = "16.0"
platforms: [.iOS(.v16)]
```

If the package has no `.platforms` declaration, no change is needed.

**Step 2 — Set `exportToKotlin` (always required)**
Run the detection steps in `references/exporting.md` § "Detecting ObjC Compatibility via the Modulemap". Never assume based on package name:
- **Confirmed ObjC-compatible** → `exportToKotlin = true`, bridge file may be empty
- **Pure Swift** → `exportToKotlin = false`, must wrap in `@objcMembers` bridge class if the user asks for bridge code

**Step 3 — Check whether the dependency needs Xcode-side linking**
After the Gradle build, the plugin auto-detects static frameworks and generates a local Swift package for anything that needs Xcode linking — printing a `logger.error` message with the path. The package is always generated inside the Kotlin module directory (next to `build.gradle.kts`), named `exported<BridgeName>` — e.g. bridge name `nativeBridge` → `<module>/exportedNativeBridge/`. Binary XCFramework targets (`.binaryTarget(...)` in `Package.swift`) may be missed by auto-detection. Add them explicitly:

```kotlin
target.swiftPackageConfig("nativeBridge") {
    exportedPackageSettings {
        includeProduct = listOf("GoogleMaps")  // explicit: don't rely on auto-detection for binary targets
    }
    dependency { ... }
}
```

> Heuristic: if the package distributes XCFrameworks in its release artifacts, it's almost certainly a binary target — use `includeProduct` proactively.

---

### Remote Version

The most common case: a published release from a remote Git repository.

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.compilations.all {
            compileTaskProvider.configure {
                compilerOptions.optIn.add("kotlinx.cinterop.ExperimentalForeignApi")
            }
        }
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                remotePackageVersion(
                    url = uri("https://github.com/krzyzanowskim/CryptoSwift.git"),
                    version = "1.8.4",
                    products = {
                        add("CryptoSwift", exportToKotlin = false)  // pure Swift → bridge only
                    },
                )
            }
        }
    }
}
```

Kotlin usage:
```kotlin
import nativeBridge.CryptoSwiftBridge

val hash = CryptoSwiftBridge().md5(of = "hello")
```

> Before setting `exportToKotlin`, check this product's ObjC compatibility using the detection steps in `references/exporting.md` § "Detecting ObjC Compatibility via the Modulemap". Every product must be verified individually.

---

### Remote Branch

Use when you need the latest commit from a branch (e.g. `main`, `develop`).

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                remotePackageBranch(
                    url = uri("https://github.com/kishikawakatsumi/KeychainAccess.git"),
                    branch = "master",
                    products = {
                        add("KeychainAccess", exportToKotlin = false)
                    },
                )
            }
        }
    }
}
```

---

### Remote Commit

Use when you need to pin to a specific, reproducible commit.

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                remotePackageCommit(
                    url = uri("https://github.com/krzyzanowskim/CryptoSwift.git"),
                    revision = "729e01bc9b9dab466ac85f21fb9ee2bc1c61b258",
                    products = {
                        add("CryptoSwift", exportToKotlin = false)
                    },
                )
            }
        }
    }
}
```

---

### Local Package (source)

Use when your Swift package lives on disk (e.g. an internal library in the same repo or a sibling directory).

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                localPackage(
                    path = "${rootDir}/LocalPackages/MyAnalytics",
                    packageName = "MyAnalytics",
                    products = {
                        add("MyAnalytics", exportToKotlin = false)
                    },
                )
            }
        }
    }
}
```

---

### Local Binary (XCFramework)

Use when you have a pre-built `.xcframework` on disk (e.g. a vendor-supplied SDK).

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                localBinary(
                    path = "${rootDir}/Frameworks/VendorSDK.xcframework",
                    packageName = "VendorSDK",
                    exportToKotlin = false  // set true only after confirming ObjC headers via modulemap
                )
            }
        }
    }
}
```

---

### Remote Binary (XCFramework zip)

Use when the vendor distributes a pre-built `.xcframework` as a downloadable zip (common for closed-source SDKs).

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                remoteBinary(
                    url = uri("https://cdn.example.com/releases/VendorSDK-2.3.1.xcframework.zip"),
                    checksum = "abc123def456...",   // SHA-256 of the zip file
                    packageName = "VendorSDK",
                    exportToKotlin = false
                )
            }
        }
    }
}
```

To get the checksum:
```bash
swift package compute-checksum /path/to/VendorSDK.xcframework.zip
```

---

### Multiple Packages in One Config

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

target.swiftPackageConfig("nativeBridge") {
    minIos = "16.0"
    dependency {
        remotePackageVersion(
            url = uri("https://github.com/firebase/firebase-ios-sdk.git"),
            version = "11.8.0",
            products = {
                add("FirebaseCore", exportToKotlin = true)
                add("FirebaseAnalytics", exportToKotlin = true)
            },
        )
        remotePackageVersion(
            url = uri("https://github.com/krzyzanowskim/CryptoSwift.git"),
            version = "1.8.4",
            products = {
                add("CryptoSwift", exportToKotlin = false)
            },
        )
        localBinary(
            path = "${rootDir}/Frameworks/VendorSDK.xcframework",
            packageName = "VendorSDK",
            exportToKotlin = false
        )
    }
}
```

---

### Making a Dependency Available to the Xcode App Target

By default, bridge dependencies are not visible to your Xcode app code. To expose them:

```kotlin
target.swiftPackageConfig("nativeBridge") {
    exportedPackageSettings {
        includeProduct = listOf("FirebaseCore", "KeychainAccess")
    }
    dependency { ... }
}
```

The plugin generates a local Swift package at build time inside the Kotlin module directory (next to `build.gradle.kts`), at `<module>/exported<BridgeName>/` — e.g. `shared/exportedNativeBridge/`. Add that package to your Xcode project as a local dependency.

> This is also the fix for `Undefined symbol: _OBJC_CLASS_$_...` — see `references/troubleshooting.md`.

---

## 8. Final Verification — Build and Launch

**If the user has an Xcode project**, setup is not complete until the app launches successfully on each Apple platform configured in `swiftPackageConfig`. Run the sequence below only for the platforms present in the user's config — skip any platform not declared. Stop at the first non-zero exit.

**If the user has no Xcode project** (library-only module, no app target), confirm `./gradlew :<shared-module>:linkDebugFramework<Platform><Arch>` succeeds for each configured target, state the launch step is skipped, and explain why.

### iOS / tvOS / watchOS (simulator)

```bash
# 1. Build the KMP framework — use the suffix matching your configured target:
#   iOS:     linkDebugFrameworkIosSimulatorArm64
#   tvOS:    linkDebugFrameworkTvosSimulatorArm64
#   watchOS: linkDebugFrameworkWatchosSimulatorArm64
./gradlew :<shared-module>:linkDebugFramework<Platform>SimulatorArm64

# 2. Pick or boot a simulator for the target platform
UDID=$(xcrun simctl list devices booted | grep -m1 -oE '[0-9A-F]{8}-[0-9A-F-]{27}')
if [ -z "$UDID" ]; then
  # iOS: grep -m1 "iPhone"   tvOS: grep -m1 "Apple TV"   watchOS: grep -m1 "Apple Watch"
  UDID=$(xcrun simctl list devices available | grep -m1 "iPhone " | grep -oE '[0-9A-F]{8}-[0-9A-F-]{27}')
  xcrun simctl boot "$UDID"
fi
echo "Using simulator: $UDID"

# 3. Build the Xcode project (Gradle run-script phase fires automatically)
xcodebuild \
  -project iosApp/iosApp.xcodeproj \
  -scheme iosApp \
  -configuration Debug \
  -destination "platform=iOS Simulator,id=$UDID" \
  -derivedDataPath build/xcode \
  build

# 4. Locate the .app and its bundle id
APP_PATH=$(find build/xcode/Build/Products -name "*.app" -not -path "*/PlugIns/*" | head -1)
BUNDLE_ID=$(/usr/libexec/PlistBuddy -c "Print :CFBundleIdentifier" "$APP_PATH/Info.plist")

# 5. Install and launch — a non-zero PID means success
xcrun simctl install "$UDID" "$APP_PATH"
xcrun simctl launch "$UDID" "$BUNDLE_ID"

# 6. Wait 10 seconds and screenshot to confirm stability
sleep 10
xcrun simctl io "$UDID" screenshot build/xcode/launch_verify.png
```

### macOS (no simulator)

```bash
./gradlew :<shared-module>:linkDebugFrameworkMacosArm64

xcodebuild \
  -project macosApp/macosApp.xcodeproj \
  -scheme macosApp \
  -configuration Debug \
  -destination "platform=macOS" \
  -derivedDataPath build/xcode \
  build

APP_PATH=$(find build/xcode/Build/Products -name "*.app" -not -path "*/PlugIns/*" | head -1)
open "$APP_PATH"
sleep 10
```

| Symptom | Likely cause |
|---|---|
| `target '<bridge>' referenced in product '<bridge>' is empty` | Bridge folder has no `.swift` file — keep `StartYourBridgeHere.swift` or add a one-line `import Foundation` file |
| `framework not found <shared>` | `FRAMEWORK_SEARCH_PATHS` doesn't reach `<module>/build/xcode-frameworks/$(CONFIGURATION)/$(SDK_NAME)` |
| `Undefined symbol: _OBJC_CLASS_$_…` | Binary XCFramework not linked — see rule 8 in SKILL.md |
| Sandbox / "Operation not permitted" in run-script phase | `ENABLE_USER_SCRIPT_SANDBOXING` still `YES` — set to `NO` |
