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
spmForKmp = "1.9.0"   # check Gradle Plugin Portal for latest

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
    id("io.github.frankois944.spmForKmp") version "1.9.0"
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

The first sync creates `src/swift/[cinteropName]/StartYourBridgeHere.swift` — a template with examples. Delete it once you add real code, or set `spmforkmp.disableStartupFile=true` in `gradle.properties` to prevent it from being generated.

**The bridge folder must always contain at least one real `.swift` file** — even when every product uses `exportToKotlin = true` and you have no bridge code to write. Swift Package Manager rejects a target with no source files: `target '<bridgeName>' referenced in product '<bridgeName>' is empty`. A `.gitkeep` does not count. If you commit the directory under version control without bridge code, drop in a one-line `Bridge.swift`:

```swift
// src/swift/<bridgeName>/Bridge.swift
import Foundation
```

This is also what the auto-generated `StartYourBridgeHere.swift` is doing for you — keep that file (and don't add a `.gitkeep`) and the issue never appears. Only run into this when you've manually pre-created the folder structure (e.g. during a CocoaPods migration before the first Gradle sync).
