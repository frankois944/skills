# Troubleshooting Reference

## "Undefined symbol" or Native Runtime Crash

**Symptoms:**
- `Undefined symbol: _OBJC_CLASS_$_...` at build time
- `Library not loaded: @rpath/...` at runtime

**Cause:** A native dependency isn't linked into the final Xcode artifact.

**Fix:** Add the dependency to Xcode via the auto-generated local package:

```kotlin
target.swiftPackageConfig(cinteropName = "nativeBridge") {
    exportedPackageSettings {
        includeProduct = listOf("TheDependencyName")
    }
}
```

After building, the plugin prints:
```
Spm4Kmp: The following dependencies [TheDependencyName] need to be added to your Xcode project.
A local Swift package has been generated at /path/to/the/local/package
```

Add that local package to your Xcode project as a local package dependency. Once done, add `spmforkmp.hideLocalPackageMessage=true` to `gradle.properties` to suppress the message.

---

## Only `SWIFT_TYPEDEFS` Visible in Kotlin After Export

**Cause:** The package is pure Swift — it has no ObjC-compatible API. `exportToKotlin = true` produces empty stubs.

**Fix:** Write a Swift bridge that wraps the package using `@objcMembers`:

```swift
// src/swift/nativeBridge/MyWrapper.swift
import Foundation
import ThePureSwiftPackage

@objcMembers public class MyWrapper: NSObject {
    public func doThing(input: String) -> String {
        return ThePureSwiftPackage.doThing(input)
    }
}
```

Remove `exportToKotlin = true` from the product — the package is bridge-only.

---

## `error: unexpected binary name … Static libraries should be prefixed with lib`

**Cause:** Xcode 26+ added a stricter naming validation for static libraries inside XCFrameworks. Some binary SDKs (e.g. GoogleMaps) ship a `GoogleMaps.a` instead of `libGoogleMaps.a`. The spmForKmp plugin prints `ERROR FOUND WHEN EXEC` and surfaces this message in its output.

**This is a false error — ignore it.** The SPM build completes successfully despite the message, and the cinterop `.def` files are generated correctly. No action is needed.

---

## `Module '_stddef' requires feature 'found_incompatible_headers'`

**Cause:** Incompatibility between Kotlin ≤ 2.1.20 and Xcode ≥ 16.3 (iPhoneOS 18.4 SDK).

**Fix:** Upgrade to Kotlin 2.1.21+ (recommended) or downgrade to Xcode 16.2.

---

## `libswift_Concurrency.dylib not loaded` During Tests

**Cause:** Swift `async/await` requires iOS 15.0+, but KMP test binaries default to a lower minimum.

**Fix:**
```kotlin
kotlin {
    iosSimulatorArm64().binaries.getTest("debug").apply {
        freeCompilerArgs += listOf(
            "-Xoverride-konan-properties=osVersionMin.ios_simulator_arm64=16.0",
        )
    }
}
```

---

## `Failed to Store Cache Entry`

**Cause:** A known Gradle caching bug, not fixable by the plugin.

**Workaround:**
```properties
# gradle.properties
org.gradle.caching=false
org.gradle.configuration-cache=false
```

---

## IDE Is Very Slow After Adding the Plugin

**Cause:** IntelliJ/Android Studio tries to resolve `Package.swift` manifests automatically.

**Fix:** In IDE Settings → Build, Execution, Deployment → Build Tools, disable **Sync Project after changes in the build script**.

Then reclaim disk space:
```bash
rm -rf ~/Library/Caches/JetBrains/IntelliJIdea*/DerivedData
```

---

## Platform Types (`UIView`, etc.) Not Resolving in Kotlin

**Cause:** `cinterop` sometimes needs a hint to map platform types from the bridge.

**Fix:** Return `NSObject` from the bridge method and cast in Kotlin:

```swift
@objcMembers public class ViewFactory: NSObject {
    public func makeView() -> NSObject {  // NSObject, not UIView
        return UIView()
    }
    // Or create a dummy subclass to force cinterop to include the type:
    @objc public class DummyView: UIView {}
}
```

```kotlin
val view = ViewFactory().makeView() as UIView
```

---

## Issues with Multiple Configurations on the Same Target

**Risk:** Multiple `swiftPackageConfig` blocks on a single target can cause workspace conflicts and slow builds.

**Best practice:** Consolidate all dependencies into a single `swiftPackageConfig` block per target group. Use `cinteropName` to share one config across multiple targets.

---

## Debugging Tips

Enable detailed tracing:
```properties
# gradle.properties
spmforkmp.enableTracing=true
```

Traces are written to the `spmForKmpTrace` directory in your project root.
