# Swift → Kotlin Interoperability Guide

Practical patterns for calling Swift code from Kotlin via the bridge. All examples are from the official [playground repository](https://github.com/frankois944/Swift-Import-Interoperability-with-Kotlin-Multiplatform).

---

## Gradle Configuration for Advanced Features

```kotlin
// build.gradle.kts
iosTarget.swiftPackageConfig(cinteropName = "playground") {
    toolsVersion = "6.0"                          // enables Swift 6 async/await + typed throws
    strictEnums = listOf("MyEnum", "MyError")     // makes enums type-safe in Kotlin
    newPublicationInteroperability = true          // experimental: better C/ObjC interop (Kotlin 2.3.20+)
}

// Required for async/await in iOS simulator tests:
iosSimulatorArm64().binaries.getTest("debug").apply {
    freeCompilerArgs += listOf(
        "-Xoverride-konan-properties=osVersionMin.ios_simulator_arm64=16.0"
    )
}
```

---

## 1. Basic Interoperability

Covers class instantiation, methods, properties, name overrides, optional parameters.

### Swift (`Basic.swift`)
```swift
import Foundation

@objcMembers public class Basic: NSObject {

    public let readOnlyProperty: Int
    public var mutableProperty: Int = 42

    public init(someValue: Int) {
        self.readOnlyProperty = someValue
    }

    public override init() {
        self.readOnlyProperty = 0
    }

    deinit { print("Basic deinit") }

    public func method() { print("method") }

    public static func staticMethod() { print("staticMethod") }

    public class func classMethod() { print("classMethod") }

    // Custom @objc name override
    @objc(methodWithNameOverride)
    public func methodNameOverridden() -> String { return "HelloWorld!" }

    public func method(param: String) -> String { return "Hello \(param)!" }

    @objc(methodNumberWithParam:)
    public func methodNumber(param: NSNumber) { print("number: \(param)") }

    public func methodOptional(param: String?) { print("optional: \(param ?? "nil")") }
}
```

### Kotlin
```kotlin
import playground.Basic

// Custom initializer
val obj = Basic(someValue = 42)

// Properties
val readOnly = obj.readOnlyProperty()
obj.setMutableProperty(10)

// Instance, static, class methods
obj.method()
Basic.staticMethod()
Basic.classMethod()

// @objc name override — called by the Obj-C name, not the Swift name
obj.methodWithNameOverride()

// Parameterized method
val greeting = obj.method(param = "World")  // → "Hello World!"

// NSNumber
obj.methodNumber(param = NSNumber(42))

// Optional — pass null for nil
obj.methodOptional(param = null)
```

---

## 2. Enums

Only `Int`-based `@objc` enums can cross the boundary.

### Swift (`Enums.swift`)
```swift
import Foundation

// Int-backed enum required for ObjC/Kotlin compatibility
@objc public enum EnumData: Int {
    case value1, value2
    case value3 = 42
}

// Enum conforming to Error (usable with throws)
@objc public enum EnumError: Int, Error {
    case noError, error1
    case error2 = 42
}

// Not in strictEnums → non-type-safe in Kotlin (raw Int)
@objc public enum DefaultEnum: Int {
    case defaultValue1, defaultValue2
}

@objcMembers public class Enums: NSObject {
    public var myEnum: EnumData = .value1
    public let defaultEnum: DefaultEnum = .defaultValue1

    public func doSomething() -> EnumError { return .noError }
}
```

### Kotlin
```kotlin
import playground.EnumData
import playground.EnumError
import playground.DefaultEnumDefaultValue1   // non-type-safe: imported as raw constant
import playground.Enums

val obj = Enums()

// With strictEnums = listOf("EnumData", "EnumError") in build.gradle.kts:
// EnumData and EnumError become proper Kotlin enum classes
obj.setMyEnum(EnumData.EnumDataValue2)
val enumVal: EnumData = obj.myEnum()                   // → EnumData.EnumDataValue1

val result: EnumError = obj.doSomething()              // → EnumError.EnumErrorNoError

// Without strictEnums — DefaultEnum comes in as raw Int constants
val raw: Int = DefaultEnumDefaultValue1                // → 0
```

> **Naming convention:** `@objc public enum EnumData` with case `value1` becomes `EnumData.EnumDataValue1` in Kotlin (enum name prefixed to each case).

---

## 3. Error Handling (Swift `throws` → Kotlin exceptions)

Swift `throws` is bridged as an Objective-C `NSError**` out-parameter. Use the `executeWithErrorHandling` helper to convert it to idiomatic Kotlin exceptions.

### Swift (`Throws.swift`)
```swift
import Foundation

@objc public enum ThrowError: Int, Error {
    case noError, error1
    case error2 = 42
}

@objcMembers public class Throw: NSObject {

    // Workaround: property forces cinterop to export ThrowError type to Kotlin
    public let bindThrowErrorTypeToKotlin: ThrowError = .noError

    // Use untyped `throws` — typed throws are not yet bridgeable to ObjC
    public func justThrowAnError() throws {
        throw ThrowError.error1
    }

    public func throwAnErrorWithParamAndReturn(param: String) throws -> String {
        if param.isEmpty { throw ThrowError.error2 }
        return "Hello \(param)!"
    }
}
```

### Kotlin — `executeWithErrorHandling` utility

Add this utility once to your project (e.g. `util/SwiftErrorHandling.kt`):

```kotlin
import kotlinx.cinterop.*
import platform.Foundation.NSError

class SwiftException(val nsError: NSError) : Exception(nsError.localizedDescription)

@OptIn(ExperimentalForeignApi::class, BetaInteropApi::class)
@Throws(SwiftException::class)
fun <T> executeWithErrorHandling(
    operation: (errorPtr: CPointer<ObjCObjectVar<NSError?>>) -> T
): T = memScoped {
    val errorPtr = alloc<ObjCObjectVar<NSError?>>().ptr
    val result = operation(errorPtr)
    val error = errorPtr.pointed.value
    if (error != null) throw SwiftException(error)
    result
}
```

### Kotlin — calling throwing Swift methods
```kotlin
import playground.Throw
import playground.ThrowError

val throwObj = Throw()

// Method that always throws
try {
    executeWithErrorHandling { errorPtr ->
        throwObj.justThrowAnError(errorPtr)
    }
} catch (e: SwiftException) {
    println(e.nsError.localizedDescription)   // ThrowError.error1
}

// Method with param + return value
val result: String? = try {
    executeWithErrorHandling { errorPtr ->
        throwObj.throwAnErrorWithParamAndReturn(param = "France", error = errorPtr)
    }
} catch (e: SwiftException) {
    null
}
// result == "Hello France!"
```

---

## 4. Async/Await

Swift `async` functions cannot be called directly from Kotlin as coroutines. Bridge them using completion handlers (`@objc(name::)` async renamed) and wrap in `suspendCoroutine`.

### Swift (`Concurrency.swift`)
```swift
import Foundation

@objc public enum ConcurrencyError: Int, Error {
    case noError = -1
    case invalidData = 4503
    case taskCancel
}

@available(iOS 16.0.0, *)
@objcMembers public class Concurrency: NSObject {

    // Forces cinterop to export ConcurrencyError type
    public let bindConcurrencyErrorTypeToKotlin: ConcurrencyError = .noError

    // Void async
    @objc(doAsyncStuff:)
    public func doAsyncStuff() async {
        try? await Task.sleep(for: .seconds(0.5))
    }

    // Static async
    @objc(doAsyncWithStuffWithStaticMethod:)
    public static func doAsyncWithStuffWithStaticMethod() async {
        try? await Task.sleep(for: .seconds(0.5))
    }

    // Async + return value
    @objc(doAsyncStuffWithValue::)
    public func doAsyncStuffWithValue(input: String) async -> String {
        try? await Task.sleep(for: .seconds(0.5))
        return "Hello \(input)!"
    }

    // Async + return value + throws → bridged with callback (result, error)
    @objc(doAsyncStuffWithValueAndError::)
    public func doAsyncStuffWithValueAndError(input: String) async throws -> String {
        if input.isEmpty { throw ConcurrencyError.invalidData }
        return "Hello \(input)!"
    }

    // Callback-based (no async keyword) — most compatible with Kotlin
    @MainActor
    @objc(doAsyncStuffWithoutAsync::)
    public func doAsyncStuffWithoutAsync(
        input: String,
        onResult: @escaping @Sendable (String?, NSError?) -> Void
    ) {
        DispatchQueue.global().async {
            if input.isEmpty {
                onResult(nil, NSError(domain: "\(ConcurrencyError.invalidData)",
                                     code: ConcurrencyError.invalidData.rawValue))
            } else {
                onResult("Hello \(input)!", nil)
            }
        }
    }
}
```

### Kotlin

> Requires `toolsVersion = "6.0"` in `swiftPackageConfig` and `osVersionMin` override for simulator tests.

```kotlin
import playground.Concurrency
import kotlinx.coroutines.test.runTest
import kotlin.coroutines.resume
import kotlin.coroutines.resumeWithException
import kotlin.coroutines.suspendCoroutine

val obj = Concurrency()

// Void async → suspendCoroutine
runTest {
    suspendCoroutine { cont ->
        obj.doAsyncStuff { cont.resume(Unit) }
    }
}

// Static async
runTest {
    suspendCoroutine { cont ->
        Concurrency.doAsyncWithStuffWithStaticMethod { cont.resume(Unit) }
    }
}

// Async with return value
runTest {
    val result: String? = suspendCoroutine { cont ->
        obj.doAsyncStuffWithValue("France") { value -> cont.resume(value) }
    }
    // result == "Hello France!"
}

// Async throws — callback receives (result, error)
runTest {
    val result: String? = try {
        suspendCoroutine { cont ->
            obj.doAsyncStuffWithValueAndError("France") { value, error ->
                if (error != null) cont.resumeWithException(SwiftException(error))
                else cont.resume(value)
            }
        }
    } catch (e: SwiftException) { null }
}

// Callback-based (same wrapping pattern, works without async keyword)
runTest {
    val result: String? = suspendCoroutine { cont ->
        obj.doAsyncStuffWithoutAsync("France") { value, error ->
            if (error != null) cont.resumeWithException(SwiftException(error))
            else cont.resume(value)
        }
    }
    // result == "Hello France!"
}
```

---

## 5. Wrapping Pure Swift Types (Structs / Third-Party APIs)

Pure Swift structs cannot cross the ObjC boundary. Wrap them in an `@objcMembers` class.

### Swift (`ObjectWrapper.swift`)
```swift
import Foundation

// Pure Swift struct — NOT accessible from Kotlin directly
struct NativeContent {
    var message: String
    // Could also be: CoreData objects, CoreNFC, or any pure-Swift third-party type
}

// ObjC-compatible wrapper
@objcMembers public class BridgedNativeContent: NSObject {
    var nativeObject: NativeContent

    init(nativeObject: NativeContent) {
        self.nativeObject = nativeObject
        super.init()
    }

    // Expose the wrapped data via ObjC-compatible computed property
    @objc public var stringValue: String {
        get { nativeObject.message }
        set { nativeObject.message = newValue }
    }
}

// Static factory / manipulation entry point for Kotlin
@objcMembers public class ObjectWrapper: NSObject {

    @objc(getBridgedNativeClass)
    public static func getBridgedNativeClass() -> BridgedNativeContent {
        BridgedNativeContent(nativeObject: .init(message: "Hello world!"))
    }

    @objc(callNativeAPI:)
    public static func callNativeAPI(instance: BridgedNativeContent) {
        instance.nativeObject.message = String(instance.nativeObject.message.prefix(5))
    }
}
```

### Kotlin
```kotlin
import playground.ObjectWrapper
import playground.BridgedNativeContent

val wrapped: BridgedNativeContent = ObjectWrapper.getBridgedNativeClass()
println(wrapped.stringValue())          // "Hello world!"

wrapped.setStringValue("Hello me!")
println(wrapped.stringValue())          // "Hello me!"

ObjectWrapper.callNativeAPI(wrapped)
println(wrapped.stringValue())          // "Hello"
```

---

## Key Rules Summary

| What | Rule |
|---|---|
| Classes | Must extend `NSObject`, annotate with `@objcMembers public class` |
| Enums | Must be `@objc public enum MyEnum: Int` — only `Int` raw type works |
| Enums (type-safe) | Add to `strictEnums = listOf(...)` in `swiftPackageConfig` |
| Async functions | Bridge with `@objc(name:)` callback form; wrap in `suspendCoroutine` in Kotlin |
| Throwing functions | Use `executeWithErrorHandling` helper; wraps `NSError**` → `SwiftException` |
| Pure Swift structs | Wrap in an `@objcMembers class : NSObject` before exposing to Kotlin |
| Typed throws | Not bridgeable to ObjC — use untyped `throws` in bridge code |
| `struct`, `actor`, `protocol` | Cannot cross the bridge — must be wrapped or re-exposed via a class |
| Export type hint | If cinterop misses an enum/type, add a property of that type to a class: `public let bind: MyEnum = .default` |
