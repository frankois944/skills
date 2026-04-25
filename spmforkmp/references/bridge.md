# Recipe: Basic Integration (No External Dependencies)

Use this when the user only needs to call Apple SDK APIs (UIKit, Foundation, CoreLocation, AVFoundation, etc.) from Kotlin — no third-party Swift packages required.

> **Advanced interop patterns** (enums, async/await, Swift `throws`, wrapping pure Swift structs): see [`interoperability.md`](interoperability.md).

---

## Complete Example

### 1. `gradle.properties`

```properties
kotlin.mpp.enableCInteropCommonization=true
```

### 2. `build.gradle.kts`

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

plugins {
    id("org.jetbrains.kotlin.multiplatform")
    id("io.github.frankois944.spmForKmp") version "1.9.0"
}

kotlin {
    listOf(
        iosArm64(),
        iosSimulatorArm64(),
        // add macosArm64(), tvosArm64(), watchosArm64() as needed
    ).forEach { target ->
        target.compilations.all {
            compileTaskProvider.configure {
                compilerOptions.optIn.add("kotlinx.cinterop.ExperimentalForeignApi")
            }
        }
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"   // set your actual minimum
            // no dependency {} block — Apple SDK only
        }
    }
}
```

> **Single target only?** Use without a loop:
> ```kotlin
> import io.github.frankois944.spmForKmp.swiftPackageConfig
>
> iosSimulatorArm64 {
>     compilerOptions.optIn.add("kotlinx.cinterop.ExperimentalForeignApi")
>     swiftPackageConfig("iosBridge") {
>         minIos = "16.0"
>     }
> }
> ```

### 3. Swift Bridge File

Create the file at `src/swift/nativeBridge/MyBridge.swift` (the folder name matches the string passed to `swiftPackageConfig`):

```swift
import Foundation
import UIKit          // import whatever Apple frameworks you need

@objcMembers public class MyBridge: NSObject {

    // Expose a simple value
    public func appVersion() -> String {
        return Bundle.main.infoDictionary?["CFBundleShortVersionString"] as? String ?? "unknown"
    }

    // Return a platform type — use NSObject and cast in Kotlin
    public func makeLabel(text: String) -> NSObject {
        let label = UILabel()
        label.text = text
        return label
    }

    // Static method
    public static func currentLocale() -> String {
        return Locale.current.identifier
    }
}
```

**ObjC compatibility rules:**
- Class must extend `NSObject` and be `@objcMembers public class`
- All methods and properties must use `public`
- Return types must be ObjC-representable (`String`, `Int`, `Bool`, `NSObject` subclasses)
- `struct`, `enum` (with associated values), `async/await`, and Swift-only protocols cannot cross the boundary

### 4. Kotlin Usage

```kotlin
import nativeBridge.MyBridge    // module name = the string passed to swiftPackageConfig

// Instance method
val version = MyBridge().appVersion()

// Platform type — import and cast
import platform.UIKit.UILabel
val label = MyBridge().makeLabel(text = "Hello") as UILabel

// Static method
val locale = MyBridge.currentLocale()
```

---

## Returning Platform Types Safely

`cinterop` occasionally needs a hint to map platform types. Returning `NSObject` and casting in Kotlin is the most reliable approach:

```swift
// In Swift: return NSObject
public func makeView() -> NSObject {
    return UIView()
}

// Or create a dummy subclass to force cinterop to include the type:
@objcMembers public class BridgeUIView: UIView {}
```

```kotlin
import platform.UIKit.UIView
val view = MyBridge().makeView() as UIView
```

---

## Bundling Resources (v1.9.0+)

To bundle files (images, JSON, fonts) alongside your bridge:

| Folder (inside bridge dir) | SPM rule | Access in Swift |
|---|---|---|
| `Resources-process/` | `.process` (recommended) | `Bundle.module.url(forResource:withExtension:)` |
| `Resources-copy/` | `.copy` | Same |
| `Resources-embed/` | `.embedInCode` | Same |

```swift
let url = Bundle.module.url(forResource: "config", withExtension: "json")
```

---

## Performance Tips

Keep the SPM working directory outside the Gradle build folder to survive `./gradlew clean`:

```kotlin
target.swiftPackageConfig("nativeBridge") {
    spmWorkingPath = "${projectDir}/SPM"
}
```

Cache this directory in CI/CD to avoid redundant Swift compilations.
