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
                        add("FirebaseCore", "FirebaseAnalytics", exportToKotlin = true)
                    },
                )
                remotePackageVersion(
                    url = uri("https://github.com/krzyzanowskim/CryptoSwift.git"),
                    version = "1.2.3",
                    products = {
                        add("CryptoSwift")  // pure Swift, used in bridge only
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

### Remove CocoaPods integration

```bash
cd ios/  # your iOS app folder
pod deintegrate
```

You can also delete `Podfile`, `Podfile.lock`, and the `.xcworkspace` if you were using one.

### Add a new Run Script build phase

In Xcode → your target → **Build Phases**:

1. Click **+** → **New Run Script Phase**
2. Drag it to the **top** of the phase list (before "Compile Sources")
3. Paste:

```bash
cd "$SRCROOT/../../"
./gradlew :shared:embedAndSignAppleFrameworkForXcode
```

Adjust the path to match your project structure (the `cd` path should reach the directory containing `gradlew`).

### Update Build Settings

| Setting | Value |
|---|---|
| **Other Linker Flags** | `-framework shared` |
| **Framework Search Paths** | `$(SRCROOT)/../build/xcode-frameworks/$(CONFIGURATION)/$(SDK_NAME)` |

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

If you had Swift bridge code before, move it there. For ObjC-compatible packages like Firebase, the bridge file can be empty.

## Mapping CocoaPods Concepts to spmForKmp

| CocoaPods | spmForKmp |
|---|---|
| `pod("Name") { version = "x.y.z" }` | `remotePackageVersion(url = ..., version = "x.y.z", products = { add("Name") })` |
| `pod("Name") { linkOnly = true }` | `add("Name")` without `exportToKotlin` |
| `ios.deploymentTarget = "16.0"` | `minIos = "16.0"` |
| Import prefix `cocoapods.FirebaseCore.FIRApp` | `FirebaseCore.FIRApp` (or use `packageDependencyPrefix`) |
