# Migrating from the KMP CocoaPods Plugin

The Kotlin CocoaPods plugin (`org.jetbrains.kotlin.native.cocoapods`) is deprecated. This guide covers the full migration to `spmForKmp`.

## 1. Update `libs.versions.toml`

```toml
[versions]
spmForKmp = "1.9.0"   # check Gradle Plugin Portal for latest

[plugins]
# Remove:
# kotlinCocoapods = { id = "org.jetbrains.kotlin.native.cocoapods", version.ref = "kotlin" }
# Add:
spmForKmp = { id = "io.github.frankois944.spmForKmp", version.ref = "spmForKmp" }
```

## 2. Root `build.gradle.kts`

```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform).apply(false)
    // Remove: alias(libs.plugins.kotlinCocoapods).apply(false)
    alias(libs.plugins.spmForKmp).apply(false)  // add
}
```

## 3. Module `build.gradle.kts`

Replace the entire `cocoapods {}` block with `swiftPackageConfig`.

**Before:**
```kotlin
plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.kotlinCocoapods)  // remove
}

kotlin {
    iosArm64()
    iosSimulatorArm64()

    cocoapods {
        summary = "Kotlin module with cocoapods dependencies"
        ios.deploymentTarget = "16.6"

        pod("FirebaseCore") { version = libs.versions.firebaseIOS.get() }
        pod("FirebaseAnalytics") { version = libs.versions.firebaseIOS.get() }
        pod("CryptoSwift") { version = "1.2.3"; linkOnly = true }
    }
}
```

**After:**
```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

plugins {
    alias(libs.plugins.kotlinMultiplatform)
    alias(libs.plugins.spmForKmp)  // add
}

kotlin {
    listOf(
        iosArm64(),
        iosSimulatorArm64(),
    ).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.6"
            dependency {
                remotePackageVersion(
                    url = uri("https://github.com/firebase/firebase-ios-sdk.git"),
                    version = libs.versions.firebaseIOS.get(),
                    products = {
                        add("FirebaseCore", exportToKotlin = true)      // verified: has ObjC headers
                        add("FirebaseAnalytics", exportToKotlin = true) // verified: has ObjC headers
                    },
                )
                remotePackageVersion(
                    url = uri("https://github.com/krzyzanowskim/CryptoSwift.git"),
                    version = "1.2.3",
                    products = {
                        add("CryptoSwift", exportToKotlin = false)  // pure Swift → bridge only
                    },
                )
            }
        }
    }
}
```

## 4. `gradle.properties`

```properties
kotlin.mpp.enableCInteropCommonization=true
```

## 5. Xcode Changes

**Always edit the existing `.xcodeproj` — never generate a new one.** The existing project contains signing configuration, capabilities, entitlements, and other settings that are hard to reconstruct. Make surgical edits to the existing `project.pbxproj` or use Xcode's UI; do not rewrite it from scratch.

### Remove CocoaPods integration

To ensure a clean migration, you must remove CocoaPods from your existing Xcode project. **Use `pod deintegrate` in priority** as it is the cleanest and most reliable method for updating the `project.pbxproj` file in place.

#### Option A: Using `pod deintegrate` (Recommended)

Run this inside your iOS app folder:

```bash
cd ios/  # your iOS app folder
pod deintegrate
rm -f Podfile Podfile.lock
```

`pod deintegrate` removes all CocoaPods build phases, xcconfig references, and framework entries from the **existing** `project.pbxproj` in place. 

> **About `.xcworkspace`**: If your `.xcworkspace` was created by CocoaPods (the most common case), you should delete it and open the `.xcodeproj` from now on. However, if you use the workspace for other projects or custom configurations, **keep it** and simply stop using the CocoaPods-generated schemes.

#### Option B: Manual Removal (If `pod` command is unavailable)

If you cannot run `pod deintegrate`, follow these steps to manually strip CocoaPods references from your `.xcodeproj`:

1. **Delete files**: `rm -f Podfile Podfile.lock`. Delete the `.xcworkspace` **only if** it was used exclusively for CocoaPods.
2. **Open `.xcodeproj`**: Use Xcode to open your project directly (or the workspace if you decided to keep it).
3. **Remove Build Phases**: In your app target's **Build Phases**, delete any phase starting with `[CP]`, such as:
    - `[CP] Check Pods Manifest.lock`
    - `[CP] Copy Pods Resources`
    - `[CP] Embed Pods Frameworks`
4. **Remove Frameworks**: In **Build Phases** → **Link Binary With Libraries**, remove any `libPods-*.a` or `Pods_*.framework` entries.
5. **Reset Configurations**: In the Project settings → **Info** tab → **Configurations**, set the "Based on Configuration File" setting to **None** for all targets and configurations.
6. **Cleanup Search Paths**: In **Build Settings**, search for `PODS_` and clear any CocoaPods-specific values from **Framework Search Paths**, **Library Search Paths**, and **User Header Search Paths**.

### Add a new Run Script build phase

In Xcode → your target → **Build Phases**:

1. Click **+** → **New Run Script Phase**
2. Drag it to the **top** of the phase list (before "Compile Sources")
3. Paste:

```bash
cd "$SRCROOT/../"
./gradlew :shared:embedAndSignAppleFrameworkForXcode
```

Adjust the path to match your project structure (the `cd` path should reach the directory containing `gradlew`). `SRCROOT` is the directory that contains the `.xcodeproj` file; going one level up (`..`) typically reaches the project root where `gradlew` lives.

### Update Build Settings

| Setting | Value |
|---|---|
| **Other Linker Flags** | `-framework shared` |
| **Framework Search Paths** | `$(SRCROOT)/../build/xcode-frameworks/$(CONFIGURATION)/$(SDK_NAME)` |
| **Enable User Script Sandboxing** | `NO` |

`ENABLE_USER_SCRIPT_SANDBOXING` **must be `NO`** for the Gradle run script phase. The script needs network access (to download SPM packages and Gradle dependencies) and write access to the build directory — both blocked by the sandbox. Without this setting, the build will silently fail or hang.

> **Why not regenerate the project?** Rewriting `project.pbxproj` from scratch drops code-signing identities, provisioning profiles, capabilities (Push, Background Modes, etc.), custom build rules, and any other settings accumulated in the existing project. Always mutate the existing file.

## 6. Update Kotlin Import Statements

The CocoaPods plugin prefixed imports with `cocoapods.`:

```kotlin
// Before:
import cocoapods.FirebaseCore.FIRApp
import cocoapods.FirebaseAnalytics.FIRAnalytics

// After:
import FirebaseCore.FIRApp
import FirebaseAnalytics.FIRAnalytics
```

To restore the prefix during transition (avoids touching all import sites at once):

```kotlin
target.swiftPackageConfig("nativeBridge") {
    packageDependencyPrefix = "cocoapods"
}
```

## 7. Create a Bridge Folder

The plugin looks for Swift files in `src/swift/[cinteropName]/`. Create the folder:

```bash
mkdir -p shared/src/swift/nativeBridge
```

If you had Swift bridge code before, move it there. For packages confirmed ObjC-compatible (verified via modulemap or `@objc`/`@objcMembers`), the bridge file can be empty.

## 8. Final Verification — Build and Launch on a Simulator

A migration is not done until the app builds in Xcode AND boots on a simulator. Run the full chain below at the project root. Stop and fix on the first non-zero exit.

### Pre-flight checklist (common gotchas)

- **Audit every dependency for binary targets BEFORE the first `xcodebuild` run.** For each pod being migrated, open the SPM equivalent's `Package.swift` and look for `.binaryTarget(...)`. If any product is binary (very common for vendor SDKs — GoogleMaps, Firebase Crashlytics, Adyen, etc.), add `exportedPackageSettings { includeProduct = listOf(...) }` to its `swiftPackageConfig` block and wire the auto-generated `<module>/exportedNativeBridge/` into `project.pbxproj` following rule 8. Skipping this audit is the #1 cause of a "green Gradle, red Xcode" migration: the bridge `.a` compiles fine, but the iOS link step fails with `Undefined symbol: _OBJC_CLASS_$_<class>` and you only catch it at the simulator launch.
- **`build/spmKmpPlugin/` cleared if a previous run failed mid-way.** Stale `scratch/artifacts/` from an aborted SPM fetch surfaces as `XCFramework Info.plist not found` or `unexpected binary name`. `rm -rf <module>/build/spmKmpPlugin` and re-run.
- **`xcodebuild -resolvePackageDependencies` runs before `xcodebuild build`** if any binary product was added via `exportedPackageSettings`. Otherwise the build fails with `Missing package product` even when the pbxproj is correct.

### Verification flow

```bash
# 1. Build the KMP framework end-to-end (sanity check before Xcode)
./gradlew :composeApp:linkDebugFrameworkIosSimulatorArm64

# 2. Pick the first booted simulator, or boot the first available iPhone if none is running
UDID=$(xcrun simctl list devices booted | grep -m1 -oE '[0-9A-F]{8}-[0-9A-F-]{27}')
if [ -z "$UDID" ]; then
  UDID=$(xcrun simctl list devices available | grep -m1 "iPhone " | grep -oE '[0-9A-F]{8}-[0-9A-F-]{27}')
  xcrun simctl boot "$UDID"
fi
echo "Using simulator: $UDID"

# 3. Build the Xcode project (the Gradle run-script phase fires automatically)
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

# 5. Install and launch
xcrun simctl install "$UDID" "$APP_PATH"
xcrun simctl launch "$UDID" "$BUNDLE_ID"

# 6. Check runtime stability
Wait 10 seconds to detect runtime issues, also take a screenshot to proof the app is running fine, if no issue so the migration is done.
```

A clean `simctl launch` that returns a PID is the success signal. If it crashes or `xcodebuild` fails, the most common causes:

| Symptom | Likely cause |
|---|---|
| `target '<bridge>' referenced in product '<bridge>' is empty` | Bridge folder has no `.swift` file — see pre-flight |
| `XCFramework Info.plist not found at …/scratch/artifacts/…` | Stale partial fetch — `rm -rf <module>/build/spmKmpPlugin` and rebuild |
| `framework not found ComposeApp` | `FRAMEWORK_SEARCH_PATHS` doesn't reach `<module>/build/xcode-frameworks/$(CONFIGURATION)/$(SDK_NAME)` |
| `Undefined symbol: _OBJC_CLASS_$_…` | Binary XCFramework not linked — see rule 8 in SKILL.md and `references/troubleshooting.md` |
| Sandbox / "Operation not permitted" during the run-script phase | `ENABLE_USER_SCRIPT_SANDBOXING` still `YES` — set to `NO` in both Debug and Release |

## Mapping CocoaPods Concepts to spmForKmp

| CocoaPods | spmForKmp |
|---|---|
| `pod("Name") { version = "x.y.z" }` | `remotePackageVersion(url = ..., version = "x.y.z", products = { add("Name") })` |
| `pod("Name") { linkOnly = true }` | `add("Name", exportToKotlin = false)` |
| `ios.deploymentTarget = "16.0"` | `minIos = "16.0"` |
| Import prefix `cocoapods.FirebaseCore.FIRApp` | `FirebaseCore.FIRApp` (or use `packageDependencyPrefix`) |
