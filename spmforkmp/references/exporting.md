# Exporting Packages to Kotlin

## When to Export vs. When to Bridge

| Scenario | Approach |
|---|---|
| Package has an ObjC-compatible API (e.g., Firebase, Alamofire) | `exportToKotlin = true` — types appear directly in Kotlin |
| Package is pure Swift (no ObjC headers) | Write a bridge wrapper with `@objcMembers` |

To check: look at the package's GitHub Languages section. "Swift" only → pure Swift. If it has Objective-C or mixed, it's likely compatible.

You can also inspect the generated module map after a build:
```
build/spmKmpPlugin/[cinteropName]/scratch/release/[Product].framework/Modules/module.modulemap
```
If the header is empty (just boilerplate), the package has no ObjC interface.

## Export a Package to Kotlin

```kotlin
target.swiftPackageConfig("nativeBridge") {
    dependency {
        remotePackageVersion(
            url = uri("https://github.com/firebase/firebase-ios-sdk.git"),
            version = "11.8.0",
            products = {
                add("FirebaseAnalytics", exportToKotlin = true)
                add("FirebaseCore", exportToKotlin = true)
                add("FirebaseMessaging")  // not exported, bridge-only
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
