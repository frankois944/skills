# Recipe: External SPM Dependency Integration

Each section below is a complete, self-contained recipe. Always produce: Gradle config + Swift bridge (or note it can be empty) + Kotlin usage.

The key choice per package:
- **ObjC-compatible** (Firebase, Alamofire, etc.) → can use `exportToKotlin = true`, bridge file may be empty
- **Pure Swift** (CryptoSwift, KeychainAccess, etc.) → must wrap in `@objcMembers` bridge class, no `exportToKotlin`

---

## Remote Version

The most common case: a published release from a remote Git repository.

### `gradle.properties`
```properties
kotlin.mpp.enableCInteropCommonization=true
```

### `build.gradle.kts`
```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

plugins {
    id("org.jetbrains.kotlin.multiplatform")
    id("io.github.frankois944.spmForKmp") version "1.9.0"
}

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.compilations.all {
            compileTaskProvider.configure {
                compilerOptions.optIn.add("kotlinx.cinterop.ExperimentalForeignApi")
            }
        }
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                remotePackageVersion(
                    url = uri("https://github.com/krzyzanowskim/CryptoSwift.git"),
                    version = "1.8.4",
                    products = {
                        add("CryptoSwift", exportToKotlin = false)  // pure Swift → bridge only
                    },
                )
            }
        }
    }
}
```

### `src/swift/nativeBridge/CryptoSwiftBridge.swift`
```swift
import Foundation
import CryptoSwift

@objcMembers public class CryptoSwiftBridge: NSObject {
    public func md5(of value: String) -> String {
        return value.md5()
    }
    public func sha256(of value: String) -> String {
        return value.sha256()
    }
}
```

### Kotlin
```kotlin
import nativeBridge.CryptoSwiftBridge

val hash = CryptoSwiftBridge().md5(of = "hello")
```

> **ObjC-compatible package?** Use `exportToKotlin = true` and leave the bridge file empty:
> ```kotlin
> add("FirebaseAnalytics", exportToKotlin = true)
> ```
> See `references/exporting.md` for the complete export recipe.

---

## Remote Branch

Use when you need the latest commit from a branch (e.g. `main`, `develop`).

### `build.gradle.kts`
```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                remotePackageBranch(
                    url = uri("https://github.com/kishikawakatsumi/KeychainAccess.git"),
                    branch = "master",
                    products = {
                        add("KeychainAccess", exportToKotlin = false)  // pure Swift → bridge only
                    },
                )
            }
        }
    }
}
```

### `src/swift/nativeBridge/KeychainBridge.swift`
```swift
import Foundation
import KeychainAccess

@objcMembers public class KeychainBridge: NSObject {
    private let keychain: Keychain

    public init(service: String) {
        self.keychain = Keychain(service: service)
    }

    public func set(value: String, forKey key: String) -> Bool {
        do { try keychain.set(value, key: key); return true }
        catch { return false }
    }

    public func get(forKey key: String) -> String? {
        return try? keychain.get(key)
    }

    public func remove(forKey key: String) -> Bool {
        do { try keychain.remove(key); return true }
        catch { return false }
    }
}
```

### Kotlin
```kotlin
import nativeBridge.KeychainBridge

val kc = KeychainBridge(service = "com.example.app")
kc.set(value = "secret", forKey = "token")
val token: String? = kc.get(forKey = "token")
```

---

## Remote Commit

Use when you need to pin to a specific, reproducible commit (useful in CI or when a version tag doesn't exist yet).

### `build.gradle.kts`
```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                remotePackageCommit(
                    url = uri("https://github.com/krzyzanowskim/CryptoSwift.git"),
                    revision = "729e01bc9b9dab466ac85f21fb9ee2bc1c61b258",
                    products = {
                        add("CryptoSwift", exportToKotlin = false)  // pure Swift → bridge only
                    },
                )
            }
        }
    }
}
```

The bridge Swift file and Kotlin usage are identical to the Remote Version example above.

---

## Local Package (source)

Use when your Swift package lives on disk (e.g. an internal library in the same repo or a sibling directory).

### `build.gradle.kts`
```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                localPackage(
                    path = "${rootDir}/LocalPackages/MyAnalytics",   // absolute path to the Swift package folder
                    packageName = "MyAnalytics",
                    products = {
                        add("MyAnalytics", exportToKotlin = false)  // set true only after confirming ObjC-compatible
                    },
                )
            }
        }
    }
}
```

### `src/swift/nativeBridge/AnalyticsBridge.swift`
```swift
import Foundation
import MyAnalytics

@objcMembers public class AnalyticsBridge: NSObject {
    public static func track(event: String) {
        MyAnalytics.track(event)
    }
}
```

### Kotlin
```kotlin
import nativeBridge.AnalyticsBridge

AnalyticsBridge.track(event = "app_open")
```

---

## Local Binary (XCFramework)

Use when you have a pre-built `.xcframework` on disk (e.g. a vendor-supplied SDK).

### `build.gradle.kts`
```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                localBinary(
                    path = "${rootDir}/Frameworks/VendorSDK.xcframework",
                    packageName = "VendorSDK",
                    exportToKotlin = false  // set true only after confirming ObjC headers via modulemap
                )
            }
        }
    }
}
```

### `src/swift/nativeBridge/VendorBridge.swift`

If the XCFramework has pure Swift API, wrap it:
```swift
import Foundation
import VendorSDK

@objcMembers public class VendorBridge: NSObject {
    public static func initialize(apiKey: String) {
        VendorSDK.setup(apiKey: apiKey)
    }
}
```

If it has ObjC headers, use `exportToKotlin = true` and leave the bridge file empty.

### Kotlin
```kotlin
import nativeBridge.VendorBridge

VendorBridge.initialize(apiKey = "YOUR_KEY")
```

---

## Remote Binary (XCFramework zip)

Use when the vendor distributes a pre-built `.xcframework` as a downloadable zip (common for closed-source SDKs).

### `build.gradle.kts`
```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

kotlin {
    listOf(iosArm64(), iosSimulatorArm64()).forEach { target ->
        target.swiftPackageConfig("nativeBridge") {
            minIos = "16.0"
            dependency {
                remoteBinary(
                    url = uri("https://cdn.example.com/releases/VendorSDK-2.3.1.xcframework.zip"),
                    checksum = "abc123def456...",   // SHA-256 of the zip file
                    packageName = "VendorSDK",
                    exportToKotlin = false  // set true only after confirming ObjC headers via modulemap
                )
            }
        }
    }
}
```

To get the checksum:
```bash
swift package compute-checksum /path/to/VendorSDK.xcframework.zip
```

### Bridge and Kotlin usage

Same pattern as Local Binary above.

---

## Multiple Packages in One Config

You can mix dependency types in a single `dependency {}` block:

```kotlin
import io.github.frankois944.spmForKmp.swiftPackageConfig

target.swiftPackageConfig("nativeBridge") {
    minIos = "16.0"
    dependency {
        remotePackageVersion(
            url = uri("https://github.com/firebase/firebase-ios-sdk.git"),
            version = "11.8.0",
            products = {
                add("FirebaseCore", exportToKotlin = true)    // ObjC → export
                add("FirebaseAnalytics", exportToKotlin = true)
            },
        )
        remotePackageVersion(
            url = uri("https://github.com/krzyzanowskim/CryptoSwift.git"),
            version = "1.8.4",
            products = {
                add("CryptoSwift", exportToKotlin = false)   // pure Swift → bridge only
            },
        )
        localBinary(
            path = "${rootDir}/Frameworks/VendorSDK.xcframework",
            packageName = "VendorSDK",
            exportToKotlin = false  // confirm ObjC-compatibility before changing
        )
    }
}
```

---

## Making a Dependency Available to the Xcode App Target

By default, bridge dependencies are not visible to your Xcode app code. To expose them:

```kotlin
target.swiftPackageConfig("nativeBridge") {
    exportedPackageSettings {
        includeProduct = listOf("FirebaseCore", "KeychainAccess")
    }
    dependency { ... }
}
```

The plugin generates a local Swift package at build time and prints its path. Add that package to your Xcode project as a local dependency.

> This is also the fix for `Undefined symbol: _OBJC_CLASS_$_...` — see `references/troubleshooting.md`.
