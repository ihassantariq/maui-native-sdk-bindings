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
    │  Swift  │   │   Java    │
    │ Wrapper │   │  Wrapper  │
    │(XCFrame)│   │   (AAR)   │
    └────┬────┘   └─────┬─────┘
         │              │
    ┌────┴────┐   ┌─────┴─────┐
    │ Native  │   │  Native   │
    │ iOS SDK │   │Android SDK│
    └─────────┘   └───────────┘
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

Android SDKs come as **AAR/JAR files**, which .NET can bind more directly. The binding generator auto-creates C# wrappers from Java bytecode.

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net9.0-android</TargetFramework>
        <SupportedOSPlatformVersion>21</SupportedOSPlatformVersion>
    </PropertyGroup>
    
    <ItemGroup>
        <PackageReference Include="Xamarin.AndroidX.Room.Common" Version="2.7.2" />
        <PackageReference Include="GoogleGson" Version="2.13.1" />
        <PackageReference Include="Square.Retrofit2" Version="3.0.0" />
    </ItemGroup>

    <ItemGroup>
        <AndroidLibrary Include="vendor-sdk.aar"/>
    </ItemGroup>
</Project>
```

### ⚠️ Metadata Transformations — The Critical Missing Piece

For complex Android SDKs, the auto-generated C# code will have **compilation errors**. You fix them with `Transforms/Metadata.xml`. Here are the most common patterns:

#### 1. Removing Conflicting Repackaged Classes
```xml
<!-- SDK bundles its own protobuf that conflicts with NuGet -->
<remove-node path="/api/package[@name='repack.com.google.protobuf']/class[@name='ByteString']" />
```

#### 2. Fixing Parcelable CREATOR Return Types
```xml
<!-- Generic type inference fails for Parcelable.Creator<T> -->
<attr path="/api/package[@name='com.vendor.health']/class[@name='Config.CREATOR']/method[@name='createFromParcel']" 
      name="managedReturn">Java.Lang.Object</attr>
```

#### 3. Removing Obfuscated Duplicate Methods
```xml
<!-- ProGuard creates methods like a(), b(), c() that collide in C# -->
<remove-node path="/api/package[@name='com.vendor.internal']/class[@name='InitArgs']/method[@name='d' and count(parameter)=0]" />
```

#### 4. Gson Serializer Type Fixes
```xml
<attr path="/api/package[@name='com.vendor.utilities']/class[@name='DateSerializer']/method[@name='serialize']"
      name="managedReturn">GoogleGson.JsonElement</attr>
```

#### 5. Kotlin Companion Object Renaming
```xml
<attr path="/api/package[@name='com.vendor.health']/class[@name='Manager']/field[@name='Companion']"
      name="managedName">CompanionInstance</attr>
```

> **💡 Process:** Start with an empty `Metadata.xml`, build the project, read the errors, add a fix, rebuild. Repeat. Budget 1–3 days for this on complex SDKs.

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
