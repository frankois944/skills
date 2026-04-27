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

**Adding the local package programmatically (CI / no Xcode UI)**

Use the `xcodeproj` Ruby gem — it modifies `project.pbxproj` safely without manual UUID juggling:

```bash
gem install xcodeproj
```

```ruby
# add_local_package.rb
# Usage: ruby add_local_package.rb
require 'xcodeproj'

PROJECT   = 'iosApp/iosApp.xcodeproj'   # path to your .xcodeproj
TARGET    = 'iosApp'                      # app target name
PKG_PATH  = '../shared/exportedNativeBridge'  # relative path from .xcodeproj to the generated package
PKG_NAME  = 'exportedNativeBridge'            # must match the bridge name with "exported" prefix

project = Xcodeproj::Project.open(PROJECT)
target  = project.targets.find { |t| t.name == TARGET }

# Skip if already added
unless project.root_object.package_references.any? { |r| r.relative_path == PKG_PATH }
  ref = project.new(Xcodeproj::Project::Object::XCLocalSwiftPackageReference)
  ref.relative_path = PKG_PATH
  project.root_object.package_references << ref

  dep = project.new(Xcodeproj::Project::Object::XCSwiftPackageProductDependency)
  dep.package      = ref
  dep.product_name = PKG_NAME
  target.package_product_dependencies << dep

  file = project.new(Xcodeproj::Project::Object::PBXBuildFile)
  file.product_ref = dep
  target.frameworks_build_phase.files << file
end

project.save
puts "Done — run xcodebuild -resolvePackageDependencies before building"
```

```bash
ruby add_local_package.rb
xcodebuild -resolvePackageDependencies -project iosApp/iosApp.xcodeproj | tail -n 50
```

Adjust `PKG_PATH` and `PKG_NAME` to match your bridge name (e.g. bridge `nativeIosBridge` → `exportedNativeIosBridge`).

---

## `unrecognized selector sent to instance` at Runtime

**Symptom:** The app builds and links successfully but crashes at runtime with `unrecognized selector sent to instance` or `+[ClassName method]: unrecognized selector`.

**Cause:** The dependency uses Objective-C categories defined in a static library. The Apple linker dead-strips categories that aren't directly referenced by symbol name, so they vanish from the final binary even though the build succeeds.

**Fix:** Add `-ObjC` to the linker options for every Apple target in `build.gradle.kts`:

```kotlin
listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
    target.binaries.all {
        linkerOpts("-ObjC")
    }
    target.swiftPackageConfig("nativeBridge") { ... }
}
```

This forces the linker to load all ObjC categories from static libraries, not just those reachable by direct symbol reference. Firebase is the most common package that requires this flag.

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
