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
A local Swift package has been generated at <module>/exported<BridgeName>
```

Add that local package to your Xcode project as a local package dependency. Once done, add `spmforkmp.hideLocalPackageMessage=true` to `gradle.properties` to suppress the message.

**Adding the local package to `project.pbxproj` manually**

When editing the pbxproj by hand, three entries are required. Using the generated package at `../google-maps/exportedNativeBridge` as an example:

1. Define the local package reference object (any UUID):
```
/* Begin XCLocalSwiftPackageReference section */
    AABBCC001122334455667701 /* XCLocalSwiftPackageReference "../google-maps/exportedNativeBridge" */ = {
        isa = XCLocalSwiftPackageReference;
        relativePath = ../google-maps/exportedNativeBridge;
    };
/* End XCLocalSwiftPackageReference section */
```

2. Add it to `packageReferences` inside `PBXProject` — **not** `localPackages` (that key is unrecognised by xcodebuild):
```
packageReferences = (
    AABBCC001122334455667701 /* XCLocalSwiftPackageReference "../google-maps/exportedNativeBridge" */,
);
```

3. Add the product dependency with a `package` back-reference, and link it in `PBXFrameworksBuildPhase`:
```
/* XCSwiftPackageProductDependency */
AABBCC001122334455667702 /* exportedNativeBridge */ = {
    isa = XCSwiftPackageProductDependency;
    package = AABBCC001122334455667701 /* XCLocalSwiftPackageReference "..." */;
    productName = exportedNativeBridge;
};

/* PBXBuildFile */
AABBCC001122334455667703 /* exportedNativeBridge in Frameworks */ = {
    isa = PBXBuildFile;
    productRef = AABBCC001122334455667702 /* exportedNativeBridge */;
};
```

Then add `AABBCC001122334455667702` to the target's `packageProductDependencies` and `AABBCC001122334455667703` to the `PBXFrameworksBuildPhase` files list.

The `package` field on `XCSwiftPackageProductDependency` is mandatory — omitting it leaves the product orphaned and causes `Missing package product '...'` at build time.

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

**Cause:** Xcode 26+ added stricter naming validation for static libraries inside XCFrameworks. Some binary SDKs (notably GoogleMaps `GoogleMaps_3p.xcframework`) ship a top-level `<Product>.a` instead of `lib<Product>.a`. The spmForKmp plugin prints `ERROR FOUND WHEN EXEC` and surfaces the message in its output.

**The message alone is harmless** — for the simulator slice, SPM still produces a usable scratch directory and the next cinterop task succeeds.

**But it commonly cascades on the device target** with:

```
error: XCFramework Info.plist not found at .../scratch/artifacts/<package>/<Product>/<Product>.xcframework
```

That second error happens because the simulator slice extraction aborted partway through and left an incomplete artifact tree. The device target then refuses to read it.

**Fix when the device target fails:**

1. Clear the partial fetch:
   ```bash
   rm -rf <module>/build/spmKmpPlugin
   ```

2. Build the targets one at a time (simulator first, then device), so each gets its own clean fetch:
   ```bash
   ./gradlew :<module>:SwiftPackageConfigAppleNativeBridgeCompileSwiftPackageIosSimulatorArm64
   ./gradlew :<module>:SwiftPackageConfigAppleNativeBridgeCompileSwiftPackageIosArm64
   ```

3. If the device target still fails with `Info.plist not found`, the vendor's SPM package is unusable on this Xcode version. Two fallbacks:
   - **Pin a different version.** Vendors often fix the binary layout in subsequent releases — bump the version one or two patches up or down.
   - **Bypass the broken `Package.swift` with `remoteBinary` or `localBinary`.** Download the `.xcframework.zip` directly from the vendor (for GoogleMaps: `https://dl.google.com/geosdk/swiftpm/<version>/GoogleMaps_3p.xcframework.zip`) and reference it via `localBinary(...)` after unzipping, or `remoteBinary(url, checksum, ...)` to let SPM fetch and verify it. This skips the broken `Package.swift` entirely.

> When you see `unexpected binary name`, check whether any *task* (not just a log line) failed. If only the message is present and the next task ran, ignore it. If a task FAILED with `Info.plist not found`, apply the cascade fix above.

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
