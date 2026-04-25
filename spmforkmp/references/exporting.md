# Exporting Packages to Kotlin

## When to Export vs. When to Bridge

| Scenario | Approach |
|---|---|
| Package is confirmed ObjC-compatible (verified via modulemap/headers/`@objc`) | `exportToKotlin = true` — types appear directly in Kotlin |
| Package is pure Swift (no ObjC headers, no `@objc` on public types) | `exportToKotlin = false` — write a bridge wrapper with `@objcMembers` |

Every dependency must be checked individually — never assume compatibility based on package name.

See **Detecting ObjC Compatibility via the Modulemap** below for how to determine which case applies before (or after) building.

---

## Detecting ObjC Compatibility via the Modulemap

`exportToKotlin = true` only works when a product exposes Objective-C headers. The build system follows the modulemap to find those headers — you can do the same check manually.

### Source packages (GitHub or local)

**Step 1 — read `Package.swift`**

Find the target(s) that back the product you want to export. ObjC-compatible targets either declare a `publicHeadersPath` or follow SPM's implicit convention of placing headers under `Sources/[TargetName]/include/`:

```swift
// Explicit — ObjC-compatible
.target(name: "FirebaseCore", publicHeadersPath: "Sources/Core/Public")

// Implicit — ObjC-compatible if Sources/MyTarget/include/*.h exists
.target(name: "MyTarget", ...)

// Pure Swift — no publicHeadersPath, no include/ folder
.target(name: "CryptoSwift", ...)
```

**Step 2a — inspect the ObjC header (if `include/` exists)**

Navigate to `Sources/[TargetName]/include/` (or the declared `publicHeadersPath`). Open any `.h` file and look for real ObjC declarations:

```objc
// ✅ ObjC-compatible — has real interface declarations
@interface FIRApp : NSObject
@property (class, nonatomic, readonly) FIRApp *defaultApp;
- (void)configure;
@end

// ❌ Not exportable — Swift-generated stub, no actual interface
// Only contains @class forward-declarations or typedef NS_ENUM wrappers
// with no @interface / @protocol blocks
```

If the `.h` file contains only `@class` forward-declarations and no `@interface` or `@protocol` blocks, cinterop will produce only `SWIFT_TYPEDEFS` stubs — useless from Kotlin. Use a bridge wrapper instead.

**Step 2b — scan Swift source files for `@objc` / `@objcMembers` (if no `include/` folder)**

A package can be ObjC-compatible even without hand-written `.h` files. When Swift public types are annotated with `@objc` or `@objcMembers`, the Swift compiler auto-generates an ObjC interface that cinterop can consume.

Grep the `Sources/` directory for these annotations on public declarations:

```bash
grep -rn "@objc\|@objcMembers" Sources/
```

Signs of a usable ObjC interface:

```swift
// ✅ exportToKotlin = true will work — public class exposed to ObjC
@objcMembers public class MySDKClient: NSObject {
    public func connect(url: String) { ... }
}

// ✅ also valid — individual members annotated
public class MySDKClient: NSObject {
    @objc public func connect(url: String) { ... }
    @objc public var isConnected: Bool = false
}
```

Signs it won't work:

```swift
// ❌ @objc inside a non-NSObject class — not bridgeable as-is
@objc public class PureSwiftClass {  // missing ": NSObject"
    ...
}

// ❌ @objc on an internal type — not visible to cinterop
@objc class InternalHelper: NSObject { ... }  // no `public`
```

The key requirement is `public` visibility **and** either inheriting from `NSObject` or explicit `@objc`/`@objcMembers` on each member. If the package's public API only uses structs, enums, actors, or protocols without `@objc`, it cannot be exported — wrap it in a bridge instead.

### Binary targets (XCFramework)

An XCFramework bundles a `.framework` per architecture slice. The modulemap is inside each slice:

```
VendorSDK.xcframework/
└── ios-arm64/
    └── VendorSDK.framework/
        ├── Headers/
        │   └── VendorSDK.h        ← inspect this
        └── Modules/
            └── module.modulemap   ← start here
```

**Read the modulemap first:**

```
# ✅ ObjC-compatible — umbrella header points to real ObjC headers
framework module VendorSDK {
  umbrella header "VendorSDK.h"
  export *
  module * { export * }
}

# ❌ Pure Swift stub — cinterop will produce only typedef wrappers
framework module CryptoSwift {
  header "CryptoSwift.h"
  requires objc
}
```

Then open the referenced header. Same rule: `@interface` / `@protocol` / `@property` declarations → compatible. Mostly empty or only `@class` lines → not compatible.

### After a build — check the generated modulemap

The plugin writes a `.framework` for each product into its build cache:

```
build/spmKmpPlugin/[cinteropName]/scratch/release/[Product].framework/Modules/module.modulemap
```

Open that modulemap and follow the header reference. If the header is essentially empty (boilerplate only), the product has no ObjC interface and `exportToKotlin` won't yield useful Kotlin bindings.

### Quick reference

| Signal | `exportToKotlin`? |
|---|---|
| `Sources/[Target]/include/*.h` with `@interface` blocks | ✅ yes |
| Swift source has `@objcMembers public class … : NSObject` | ✅ yes |
| Swift source has `@objc public func/var` on a public `NSObject` subclass | ✅ yes |
| No `include/` folder, no `@objc`/`@objcMembers` on public types | ❌ no — bridge wrapper required |
| `@objc` present but class doesn't inherit `NSObject` or isn't `public` | ❌ no — not bridgeable |
| Only structs / enums / actors / Swift-only protocols | ❌ no — bridge wrapper required |
| XCFramework modulemap: `umbrella header` pointing to ObjC headers | ✅ yes |
| XCFramework header: only `@class` forward-declarations, no `@interface` | ❌ no |
| Post-build header is nearly empty (Swift typedef stub) | ❌ no |

## Export a Package to Kotlin

```kotlin
target.swiftPackageConfig("nativeBridge") {
    dependency {
        remotePackageVersion(
            url = uri("https://github.com/firebase/firebase-ios-sdk.git"),
            version = "11.8.0",
            products = {
                add("FirebaseAnalytics", exportToKotlin = true)  // verified: has ObjC headers
                add("FirebaseCore", exportToKotlin = true)       // verified: has ObjC headers
                add("FirebaseMessaging", exportToKotlin = true)  // verified: has ObjC headers
            },
        )
    }
}
```

The bridge Swift file can remain empty — no wrapper needed for ObjC-compatible products.

Kotlin usage:
```kotlin
import FirebaseAnalytics.FIRConsentStatusGranted
import FirebaseCore.FIRApp

@ExperimentalForeignApi
FIRApp.configure()
val granted = FIRConsentStatusGranted
```

## Import Prefix (Optional)

If migrating from CocoaPods where imports used a `cocoapods.` prefix, or to avoid name collisions:

```kotlin
target.swiftPackageConfig("nativeBridge") {
    packageDependencyPrefix = "spm"
}
```

With prefix: `import spm.FirebaseCore.FIRApp`
Without: `import FirebaseCore.FIRApp`

## Cinterop Fine-Tuning

For packages with unusual enum layouts or ObjC initializer patterns:

```kotlin
target.swiftPackageConfig("nativeBridge") {
    // Treat specific ObjC enums as proper Kotlin enum class
    strictEnums = listOf("FIRLoggerLevel")

    // Wrap ObjC exceptions as Kotlin ForeignException
    foreignExceptionMode = "objc-wrap"

    // Additional linker flags for cinterop
    linkerOpts = listOf("-framework", "Security")
}
```
