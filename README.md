# 📱 Native SDK Bindings for .NET MAUI — A Complete Guide

> A practical, real-world guide to binding native iOS (Swift) and Android (Java/Kotlin) SDKs for use in .NET MAUI cross-platform applications.

[![.NET](https://img.shields.io/badge/.NET_9.0-512BD4?logo=dotnet&logoColor=white)](https://dotnet.microsoft.com/)
[![MAUI](https://img.shields.io/badge/.NET_MAUI-512BD4?logo=dotnet&logoColor=white)](https://learn.microsoft.com/dotnet/maui/)
[![iOS](https://img.shields.io/badge/iOS-000000?logo=apple&logoColor=white)](https://developer.apple.com/)
[![Android](https://img.shields.io/badge/Android-3DDC84?logo=android&logoColor=white)](https://developer.android.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## 🎯 What Is This?

This repository documents how to **bind native platform SDKs** (written in Swift for iOS and Java/Kotlin for Android) so they can be consumed from a **.NET MAUI** cross-platform application. 

The guide is based on a real production project where I bound a **wearable health device SDK** for use in a .NET MAUI app — enabling device scanning, pairing, health data sync (steps, heart rate, sleep, activities), and real-time updates across both iOS and Android from a single C# codebase.

## 🏗️ Architecture

```
┌─────────────────────────────────────┐
│          .NET MAUI App              │
│     (Shared UI & Business Logic)    │
└──────────────┬──────────────────────┘
               │
       ┌───────┴───────┐
       │  Platform      │
       │  Services      │
       │  (DI-injected) │
       └───┬───────┬────┘
           │       │
    ┌──────┴──┐ ┌──┴──────────┐
    │   iOS   │ │   Android   │
    │ Binding │ │   Binding   │
    │ Library │ │   Library   │
    └────┬────┘ └──────┬──────┘
         │              │
    ┌────┴────┐   ┌─────┴─────┐
    │  Swift  │   │  Native   │
    │ Wrapper │   │Android SDK│
    │(XCFrame)│   │   (AAR)   │
    └────┬────┘   └───────────┘
         │
    ┌────┴────┐
    │ Native  │
    │ iOS SDK │
    └─────────┘
```

## 📋 Table of Contents

- [iOS Binding Guide](#-ios-binding-guide)
- [Android Binding Guide](#-android-binding-guide)
- [Metadata Transformations (Android)](#-metadata-transformations--the-critical-missing-piece)
- [MAUI Integration](#-maui-integration)
- [Key Challenges & Solutions](#-key-challenges--solutions)
- [Best Practices](#-best-practices)
- [Build Instructions](#-build-instructions)

## 🍎 iOS Binding Guide

### The Problem
Most iOS SDKs are written in **Swift**, but .NET MAUI can only bind **Objective-C** APIs. We need a bridge.

### The Solution: Swift → Objective-C Wrapper → .NET Binding

**Step 1: Create a Swift wrapper** that exposes SDK functionality via `@objc` attributes:

```swift
import VendorSDK
import Foundation

@objc(WearableManagerWrapper)
public class WearableManagerWrapper: NSObject {
    
    @objc public func initializeSDK(
        license: String,
        clientId: String,
        clientSecret: String,
        completion: @escaping (Bool, String?) -> Void
    ) {
        Task {
            do {
                let config = SDKConfiguration(
                    autoReconnectWithDevices: true,
                    processData: false,
                    persistFITFiles: true
                )
                try await ConfigurationManager.shared.start(
                    withSDKConfiguration: config,
                    loggerConfiguration: loggerConfig,
                    delegate: nil
                )
                try ConfigurationManager.shared.set(license: license)
                try ConfigurationManager.shared.set(
                    clientID: clientId, secret: clientSecret
                )
                completion(true, nil)
            } catch {
                completion(false, error.localizedDescription)
            }
        }
    }
    
    @objc public func startScan() {
        Task {
            try DeviceManager.shared.scan(for: DeviceType.allCases)
        }
    }
}
```

**Key techniques:**
- `@objc` exposes Swift classes/methods to Objective-C
- `@escaping` closures replace Swift async/await for Objective-C compatibility
- Complex Swift types are simplified to primitives and NSObject subclasses

**Step 2: Build as XCFramework:**

```bash
# Build for device + simulator, then combine
xcodebuild archive -scheme WearableWrapper -sdk iphoneos \
  -archivePath ./build/Wrapper-iOS.xcarchive \
  SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES

xcodebuild archive -scheme WearableWrapper -sdk iphonesimulator \
  -archivePath ./build/Wrapper-Sim.xcarchive \
  SKIP_INSTALL=NO BUILD_LIBRARY_FOR_DISTRIBUTION=YES

xcodebuild -create-xcframework \
  -framework ./build/Wrapper-iOS.xcarchive/Products/Library/Frameworks/WearableWrapper.framework \
  -framework ./build/Wrapper-Sim.xcarchive/Products/Library/Frameworks/WearableWrapper.framework \
  -output ./build/WearableWrapper.xcframework
```

**Step 3: Create the .NET binding project** with `ApiDefinition.cs`:

```csharp
using Foundation;

namespace WearableWrapper
{
    delegate void CompletionHandler(bool success, string error);
    
    [BaseType(typeof(NSObject), Name = "WearableManagerWrapper")]
    interface WearableManagerWrapper
    {
        [Export("initializeSDKWithLicense:clientId:clientSecret:completion:")]
        void InitializeSDK(string license, string clientId, 
            string clientSecret, CompletionHandler completion);

        [Export("startScan")]
        void StartScan();

        [Export("getScannedDevices")]
        ScannedDeviceWrapper[] ScannedDevices { get; }
    }
}
```

## 🤖 Android Binding Guide

Android SDKs are distributed as **AAR** (Android Archive) or **JAR** files. Unlike iOS, you don't need to create a wrapper — .NET's binding generator can directly consume the AAR and auto-generate C# wrappers from the Java bytecode.

Here's the full process from receiving an AAR to a working binding.

### Step 1: Create a .NET Android Binding Project

```bash
dotnet new android-bindinglib -n AndroidVendorBindings
cd AndroidVendorBindings
```

This scaffolds the following structure:

```
AndroidVendorBindings/
├── AndroidVendorBindings.csproj
├── Transforms/
│   ├── Metadata.xml          # Fix binding generator issues (CRITICAL)
│   ├── EnumFields.xml        # Convert Java int constants → C# enums
│   └── EnumMethods.xml       # Convert int return types → enum types
├── Additions/                 # Add custom C# code (partial classes, helpers)
└── Jars/                      # Place AAR/JAR files here
```

### Step 2: Add the AAR/JAR Files

Copy the vendor AAR (and any companion JARs) into the `Jars/` folder and reference them in the `.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net9.0-android</TargetFramework>
        <SupportedOSPlatformVersion>21</SupportedOSPlatformVersion>
    </PropertyGroup>

    <!-- Bind the vendor SDK directly from the AAR -->
    <ItemGroup>
        <AndroidLibrary Include="Jars/vendor-sdk.aar" />
        <AndroidLibrary Include="Jars/data-processing.jar" />
    </ItemGroup>
</Project>
```

### Step 3: Identify and Add All Transitive Dependencies

This is where most people get stuck. The AAR depends on other libraries at runtime, and you must provide them as **NuGet packages**. Missing even one will cause runtime crashes.

**Option A — Use `jadx` to inspect the AAR:**

```bash
# Install jadx (Java decompiler)
brew install jadx

# Decompile the AAR to see what it imports
jadx vendor-sdk.aar -d output/

# Search for import statements to identify dependencies
grep -r "^import " output/ | sort -u
```

**Option B — Check the AAR's `pom.xml` or Gradle metadata:**

If the SDK was distributed via Maven/Gradle, check its POM file for declared dependencies. If you have the Gradle project that uses the SDK:

```bash
./gradlew dependencies --configuration releaseRuntimeClasspath
```

**Map each dependency to its NuGet equivalent:**

| Java/Gradle Dependency | NuGet Package |
|---|---|
| `androidx.room:room-common` | `Xamarin.AndroidX.Room.Common` |
| `com.google.code.gson:gson` | `GoogleGson` |
| `com.squareup.retrofit2:retrofit` | `Square.Retrofit2` |
| `com.squareup.okhttp3:okhttp` | `Square.OkHttp3` |
| `com.google.guava:guava` | `Xamarin.Google.Guava` |
| `org.jetbrains.kotlin:kotlin-stdlib` | `Xamarin.Kotlin.StdLib` |
| `androidx.core:core` | `Xamarin.AndroidX.Core` |
| `androidx.appcompat:appcompat` | `Xamarin.AndroidX.AppCompat` |

> **💡 Tip:** Search [NuGet.org](https://www.nuget.org/) for `Xamarin.AndroidX.*` or the library name. Most popular Android libraries have NuGet wrappers.

Add them to the `.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net9.0-android</TargetFramework>
        <SupportedOSPlatformVersion>21</SupportedOSPlatformVersion>
    </PropertyGroup>
    
    <!-- NuGet packages matching SDK's transitive dependencies -->
    <ItemGroup>
        <PackageReference Include="Xamarin.AndroidX.Interpolator" Version="1.0.0.33" />
        <PackageReference Include="Xamarin.AndroidX.Room.Common" Version="2.7.2" />
        <PackageReference Include="Xamarin.Google.Android.Material" Version="1.12.0.4" />
        <PackageReference Include="GoogleGson" Version="2.13.1" />
        <PackageReference Include="Square.Retrofit2" Version="3.0.0" />
        <PackageReference Include="Square.OkHttp3" Version="4.12.0" />
        <PackageReference Include="Xamarin.Kotlin.StdLib" Version="2.1.0" />
        <!-- ... match ALL transitive dependencies -->
    </ItemGroup>

    <!-- The vendor SDK AAR/JAR files -->
    <ItemGroup>
        <AndroidLibrary Include="Jars/vendor-sdk.aar" />
        <AndroidLibrary Include="Jars/data-processing.jar" />
    </ItemGroup>
</Project>
```

### Step 4: First Build — Expect Errors

```bash
dotnet build
```

Your first build **will fail**. This is normal and expected for any non-trivial SDK. The binding generator auto-creates C# code from Java bytecode in:

```
obj/Debug/net9.0-android/generated/src/
```

You can inspect these generated files to understand what went wrong. The compile errors tell you exactly what needs fixing in `Transforms/Metadata.xml`.

**Common first-build errors and what they mean:**

| Error Code | Meaning | Fix |
|---|---|---|
| `CS0433` | Duplicate type — SDK bundles a class that also exists in a NuGet | `<remove-node>` the duplicate class |
| `CS0508` | Return type mismatch — generic `Parcelable.Creator<T>` inference failed | Set `managedReturn` to `Java.Lang.Object` |
| `CS0111` | Duplicate member — obfuscated methods (`a()`, `b()`) collide | `<remove-node>` the obfuscated methods |
| `CS0535` | Interface not implemented — generic type parameter wrong | Set `managedType` on the parameter |
| `CS0534` | Abstract method not implemented — missing type mapping | Set `managedReturn` / `managedType` |

### Step 5: Fix Errors with Metadata Transformations

`Transforms/Metadata.xml` is the heart of Android binding work. It lets you modify the auto-generated API before C# compilation.

**Start with an empty file and add fixes incrementally:**

```xml
<metadata>
    <!-- Each fix goes here — rebuild after each one -->
</metadata>
```

Here are the most common transformation patterns:

#### Pattern 1: Removing Conflicting Repackaged Classes
```xml
<!-- SDK bundles its own protobuf/okhttp/etc. that conflict with NuGet -->
<remove-node path="/api/package[@name='repack.com.google.protobuf']/class[@name='ByteString']" />
<remove-node path="/api/package[@name='repack.com.google.protobuf']/class[@name='AbstractMessage']" />
<remove-node path="/api/package[@name='repack.com.google.protobuf']/class[@name='CodedOutputStream']" />
```

#### Pattern 2: Fixing Parcelable CREATOR Return Types
```xml
<!-- Generic type inference fails for Parcelable.Creator<T> -->
<attr path="/api/package[@name='com.vendor.health']/class[@name='Config.CREATOR']/method[@name='createFromParcel' and count(parameter)=1 and parameter[1][@type='android.os.Parcel']]" 
      name="managedReturn">Java.Lang.Object</attr>

<attr path="/api/package[@name='com.vendor.health']/class[@name='Config.CREATOR']/method[@name='newArray' and count(parameter)=1 and parameter[1][@type='int']]" 
      name="managedReturn">Java.Lang.Object[]</attr>
```

> **Note:** Repeat this for **every** Parcelable class in the SDK. This is the most tedious part.

#### Pattern 3: Removing Obfuscated Duplicate Methods
```xml
<!-- ProGuard/R8 creates methods like a(), b(), c() that collide in C# -->
<remove-node path="/api/package[@name='com.vendor.internal']/class[@name='InitArgs']/method[@name='d' and count(parameter)=0]" />
<remove-node path="/api/package[@name='com.vendor.internal']/class[@name='InitArgs']/method[@name='f' and count(parameter)=0]" />
```

#### Pattern 4: Fixing Gson Serializer/Deserializer Types
```xml
<attr path="/api/package[@name='com.vendor.utilities']/class[@name='DateSerializer']/method[@name='serialize' and count(parameter)=3]"
      name="managedReturn">GoogleGson.JsonElement</attr>
      
<attr path="/api/package[@name='com.vendor.utilities']/class[@name='DateSerializer']/method[@name='serialize' and count(parameter)=3]/parameter[1]"
      name="managedType">Java.Lang.Object</attr>
```

#### Pattern 5: Kotlin Companion Object Renaming
```xml
<!-- Kotlin's Companion field conflicts with C# conventions -->
<attr path="/api/package[@name='com.vendor.health']/class[@name='Manager']/field[@name='Companion']"
      name="managedName">CompanionInstance</attr>
```

#### Pattern 6: Removing Entire Problematic Packages
```xml
<!-- Remove internal/obfuscated packages you don't need -->
<remove-node path="/api/package[@name='com.vendor.internal']" />
<remove-node path="/api/package[@name='com.vendor.sdk.proguard']" />
```

### Step 6: Add ProGuard Rules

Prevent Android from stripping SDK code during release builds:

```
# proguard.cfg (add to your MAUI app project)
-keep class com.vendor.** { *; }
-keepclassmembers class com.vendor.** { *; }
-dontwarn com.vendor.**
```

### Step 7: Iterate Until Clean Build

The process is cyclical:

```
Build → Read errors → Add Metadata.xml fix → Rebuild → Repeat
```

For a complex SDK, expect **50–150+ lines** of Metadata.xml transformations. Budget **1–3 days** for this phase.

Once `dotnet build` succeeds with zero errors, your binding library is ready to consume from a MAUI project.

> **💡 Pro tip:** After a successful build, inspect `obj/Debug/net9.0-android/generated/src/` to review the final generated C# API surface. This is exactly what your MAUI app will call.

## 🔌 MAUI Integration

Define a shared interface and implement per-platform:

```csharp
// Shared interface
public interface IWearableHealthService
{
    Task<bool> Initialize(string license, string clientId, string clientSecret);
    void StartScan();
    List<WellnessData> GetWellnessData(string macAddress);
    
    event Action SdkInitialized;
    event Action<WearableDevice> DeviceConnected;
}
```

Register with dependency injection:

```csharp
// MauiProgram.cs
#if ANDROID
builder.Services.AddSingleton<IWearableHealthService, 
    Platforms.Android.WearableHealthService>();
#elif IOS
builder.Services.AddSingleton<IWearableHealthService, 
    Platforms.iOS.WearableHealthService>();
#endif
```

## 🧩 Key Challenges & Solutions

| Challenge | Problem | Solution |
|---|---|---|
| **Swift ↔ ObjC** | Swift async/await, optionals, generics don't map to ObjC | Wrap in `Task {}`, use completion handlers, simplify types |
| **Dependency Hell** | Android SDK has 20+ transitive dependencies | Match every dependency with NuGet packages, use Gradle dep tree |
| **Data Mapping** | Native models ≠ .NET models | Platform-specific mapper classes, shared POCOs |
| **Threading** | Native callbacks on background threads | `MainThread.BeginInvokeOnMainThread`, `TaskCompletionSource<T>` |
| **Obfuscated Code** | ProGuard creates duplicate method names | Remove via `Metadata.xml` — they're internal helpers |

## ✅ Best Practices

1. **Build & test native wrappers independently** before creating .NET bindings
2. **Use Objective Sharpie** to generate initial iOS API definitions
3. **Keep the wrapper API surface minimal** — only expose what you need
4. **Document every `Metadata.xml` entry** with comments explaining why
5. **Configure ProGuard rules** to prevent SDK code stripping on Android
6. **Version-lock everything** — .NET SDK, native SDK, and NuGet dependencies
7. **Use platform-specific mapper classes** at the service boundary

## 🛠️ Build Instructions

### Prerequisites
- .NET 9.0 SDK
- Xcode 15+ (for iOS)
- Android SDK, API 28+
- Java 17 (for Gradle)

### Build Steps

```bash
# 1. iOS Wrapper → XCFramework
cd WearableWrapper && ./build-xcframework.sh

# 2. Android Wrapper → AAR
cd WearableWrapperAndroid && ./gradlew assembleRelease

# 3. MAUI App
dotnet build -f net9.0-android -c Release  # Android
dotnet build -f net9.0-ios -c Release      # iOS
```

## 📚 Resources

- [.NET MAUI — Binding iOS Libraries](https://learn.microsoft.com/dotnet/maui/platform-integration/native-embedding)
- [.NET MAUI — Binding Android Libraries](https://learn.microsoft.com/xamarin/android/platform/binding-java-library/)
- [Objective Sharpie](https://learn.microsoft.com/xamarin/cross-platform/macios/binding/objective-sharpie/)
- [Android Metadata.xml Reference](https://learn.microsoft.com/xamarin/android/platform/binding-java-library/customizing-bindings/java-bindings-metadata)

## 👤 Author

**Hassan Tariq** — Senior .NET Developer, Mobile Engineer & AI/ML Specialist

- 🌐 [hassantariq.me](https://hassantariq.me)
- 💼 [Upwork — Top Rated Plus](https://www.upwork.com/freelancers/ihassantariq)
- 📧 info@hassantariq.me

---

## 📄 License

This guide is licensed under the [MIT License](LICENSE).

---

⭐ **Found this useful?** Star this repo and share it with other .NET MAUI developers!
