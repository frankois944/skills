# Setup Reference

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

## 3. Initial `swiftPackageConfig` Block

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

The first sync creates `src/swift/[cinteropName]/StartYourBridgeHere.swift` — a template with examples. Delete it once you add real code, or set `spmforkmp.disableStartupFile=true` in `gradle.properties` to prevent it from being generated. if no file inside the bridge, the plugin will generate a empty one during the build, so no need to create a dummy file.

This is also what the auto-generated `StartYourBridgeHere.swift` is doing for you — keep that file (and don't add a `.gitkeep`) and the issue never appears. Only run into this when you've manually pre-created the folder structure (e.g. during a CocoaPods migration before the first Gradle sync).

## 7. Final Verification — Build and Launch

**If the user has an Xcode project**, setup is not complete until the app launches successfully on each Apple platform configured in `swiftPackageConfig`. Run the sequence below only for the platforms present in the user's config — skip any platform not declared. Gradle BUILD SUCCESSFUL only proves the bridge compiles; Xcode build proves linking; launch proves the framework loads at runtime. Stop at the first non-zero exit.

**If the user has no Xcode project** (library-only module, no app target), the simulator launch step does not apply — state this explicitly and confirm the `linkDebugFramework…` Gradle task succeeds as the definition of done.

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
# 1. Build the KMP framework
./gradlew :<shared-module>:linkDebugFrameworkMacosArm64

# 2. Build the Xcode project
xcodebuild \
  -project macosApp/macosApp.xcodeproj \
  -scheme macosApp \
  -configuration Debug \
  -destination "platform=macOS" \
  -derivedDataPath build/xcode \
  build

# 3. Run directly and confirm no immediate crash
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
