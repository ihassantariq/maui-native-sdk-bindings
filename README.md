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

### Step 1: Create an Xcode Framework Project

1. Open **Xcode → File → New → Project**
2. Select **Framework** under the iOS tab
3. Name it `WearableWrapper` (or any name)
4. Language: **Swift**
5. Click **Create**

This gives you a framework project with the following structure:

```
WearableWrapper/
├── WearableWrapper.xcodeproj
└── WearableWrapper/
    ├── WearableWrapper.h        ← Umbrella header (auto-generated)
    ├── Info.plist
    └── (your Swift files go here)
```

### Step 2: Add the Vendor SDK as a Dependency

The vendor SDK will typically be an `.xcframework` bundle. Add it to your project:

1. Drag the `VendorSDK.xcframework` into your Xcode project
2. Make sure **"Copy items if needed"** is checked
3. In your target's **General → Frameworks and Libraries**, verify the SDK appears with **"Embed & Sign"**
4. If the SDK requires specific capabilities (e.g., Bluetooth), add them in **Signing & Capabilities**

Update `Info.plist` if the SDK requires background modes or permissions:

```xml
<key>UIBackgroundModes</key>
<array>
    <string>bluetooth-central</string>
    <string>fetch</string>
</array>
<key>NSBluetoothAlwaysUsageDescription</key>
<string>Required for communicating with wearable devices</string>
```

### Step 3: Configure the Umbrella Header

The umbrella header (`WearableWrapper.h`) is the bridge between Swift and Objective-C. Update it to expose your wrapper classes:

```objc
// WearableWrapper.h

#import <Foundation/Foundation.h>

// Auto-generated Swift-ObjC bridging header
#import "WearableWrapper-Swift.h"

// Import the vendor SDK header (if available)
#import <VendorSDK/VendorSDK.h>

// Forward-declare your wrapper classes
@class WearableManagerWrapper;
@class ScannedDeviceWrapper;
```

> **⚠️ Important:** The `-Swift.h` header is auto-generated by Xcode when you build. Don't try to create it manually.

### Step 4: Write the Swift Wrapper

Create your wrapper class that exposes SDK functionality via `@objc` attributes. The key rule: **everything exposed to .NET must be visible to Objective-C**.

```swift
// WearableWrapper.swift

import VendorSDK
import Foundation

@objc(WearableManagerWrapper)
public class WearableManagerWrapper: NSObject {
    
    // MARK: - SDK Initialization
    
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
    
    // MARK: - Device Scanning
    
    @objc public func startScan() {
        Task {
            try DeviceManager.shared.scan(for: DeviceType.allCases)
        }
    }
    
    @objc public var scannedDevices: [ScannedDeviceWrapper] {
        // Convert native SDK models → ObjC-compatible wrappers
        return nativeDevices.map { ScannedDeviceWrapper(from: $0) }
    }
}

// MARK: - ObjC-Compatible Model Wrappers

@objc(ScannedDeviceWrapper)
public class ScannedDeviceWrapper: NSObject {
    @objc public var identifier: String = ""
    @objc public var name: String = ""
    @objc public var unitId: UInt32 = 0
    
    init(from nativeDevice: NativeDevice) {
        self.identifier = nativeDevice.identifier
        self.name = nativeDevice.name
        self.unitId = nativeDevice.unitId
    }
}
```

**Key rules for ObjC compatibility:**
- Every exposed class must inherit from `NSObject`
- Use `@objc(ClassName)` to set the ObjC name explicitly
- Use `@escaping` completion handlers instead of `async/await`
- Wrap Swift async calls in `Task {}` blocks
- Only use ObjC-compatible types: `String`, `Bool`, `Int`, `NSArray`, `NSDictionary`, `NSObject` subclasses
- **Avoid:** Swift enums (unless `@objc`), generics, optionals of non-class types, tuples

### Step 5: Build as XCFramework

Build for both device and simulator architectures, then combine into a universal XCFramework:

```bash
#!/bin/bash
# build-xcframework.sh

# Clean previous builds
rm -rf ./build/WearableWrapper.xcframework

# Build for iOS devices (ARM64)
xcodebuild archive \
  -scheme WearableWrapper \
  -sdk iphoneos \
  -archivePath ./build/WearableWrapper-iOS.xcarchive \
  SKIP_INSTALL=NO \
  BUILD_LIBRARY_FOR_DISTRIBUTION=YES

# Build for iOS Simulator (x86_64 + ARM64)
xcodebuild archive \
  -scheme WearableWrapper \
  -sdk iphonesimulator \
  -archivePath ./build/WearableWrapper-Simulator.xcarchive \
  SKIP_INSTALL=NO \
  BUILD_LIBRARY_FOR_DISTRIBUTION=YES

# Combine into XCFramework
xcodebuild -create-xcframework \
  -framework ./build/WearableWrapper-iOS.xcarchive/Products/Library/Frameworks/WearableWrapper.framework \
  -framework ./build/WearableWrapper-Simulator.xcarchive/Products/Library/Frameworks/WearableWrapper.framework \
  -output ./build/WearableWrapper.xcframework

# Verify exported symbols
nm -gU build/WearableWrapper.xcframework/ios-arm64/WearableWrapper.framework/WearableWrapper | grep Wrapper
```

> **💡 Important:** `BUILD_LIBRARY_FOR_DISTRIBUTION=YES` is critical — it generates the Swift module interface, making the framework compatible across different Swift versions.

### Step 6: Generate API Definitions with Objective Sharpie

[Objective Sharpie](https://learn.microsoft.com/xamarin/cross-platform/macios/binding/objective-sharpie/) auto-generates C# binding definitions from the ObjC header:

```bash
sharpie bind \
    -sdk iphoneos26.1 \
    -output "./WearableWrapperApiDefinition" \
    -namespace "WearableWrapper" \
    -scope "build/WearableWrapper.xcframework/ios-arm64/WearableWrapper.framework/Headers/WearableWrapper-Swift.h" \
    "build/WearableWrapper.xcframework/ios-arm64/WearableWrapper.framework/Headers/WearableWrapper-Swift.h"
```

This generates `ApiDefinition.cs` and `StructsAndEnums.cs`. **You will need to clean these up manually** — Sharpie gives you ~80% of the work, but expect to fix:
- Incorrect delegate signatures
- Missing `[Export]` selectors
- Wrong property types
- Classes that need `[DisableDefaultCtor]`

### Step 7: Create the .NET iOS Binding Project

```bash
dotnet new ios-bindinglib -n iOSWearableBindings
cd iOSWearableBindings
```

Add the XCFramework references in the `.csproj`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <TargetFramework>net9.0-ios</TargetFramework>
        <IsBindingProject>true</IsBindingProject>
    </PropertyGroup>

    <ItemGroup>
        <ObjcBindingApiDefinition Include="ApiDefinition.cs" />
        <ObjcBindingCoreSource Include="StructsAndEnums.cs" />
    </ItemGroup>

    <ItemGroup>
        <!-- Your Swift wrapper framework -->
        <NativeReference Include="WearableWrapper.xcframework">
            <Kind>Framework</Kind>
            <ForceLoad>True</ForceLoad>
        </NativeReference>
        
        <!-- The vendor SDK framework -->
        <NativeReference Include="VendorSDK.xcframework">
            <Kind>Framework</Kind>
            <ForceLoad>True</ForceLoad>
        </NativeReference>
    </ItemGroup>
</Project>
```

Copy in the cleaned-up `ApiDefinition.cs` from Step 6:

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
    
    [BaseType(typeof(NSObject), Name = "ScannedDeviceWrapper")]
    [DisableDefaultCtor]
    interface ScannedDeviceWrapper
    {
        [Export("identifier")]
        string Identifier { get; set; }

        [Export("name")]
        string Name { get; set; }

        [Export("unitId")]
        uint UnitId { get; set; }
    }
}
```

Build the binding project:

```bash
dotnet build
```

If it compiles, your iOS binding is ready to reference from the MAUI project.

## 🤖 Android Binding Guide

Android SDKs are distributed as **AAR** (Android Archive) or **JAR** files. Unlike iOS, you don't need to create a wrapper — .NET's binding generator can directly consume the AAR and auto-generate C# wrappers from the Java bytecode.

**Why full SDK binding instead of a wrapper?** On Android, the binding generator handles Java/Kotlin code far more naturally than it handles Swift on iOS. By binding the entire SDK directly, you get access to the **complete API surface** without the overhead of maintaining a separate Java wrapper project. The trade-off is more `Metadata.xml` work upfront, but you avoid an extra build step and an extra layer of abstraction.

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

**🔧 Recommended: Microsoft's Xamarin Gradle Dependency Tool**

First, **create a minimal Android Library project** in Android Studio that references the vendor AAR. You don't need to write any code — this project exists solely to let Gradle resolve the SDK's dependency tree:

1. Open **Android Studio → New Project → No Activity**
2. Add a **New Module → Android Library**
3. Place the vendor AAR in a `libs/` folder
4. Add `implementation files('../libs/vendor-sdk.aar')` to the module's `build.gradle`
5. Add any dependencies the SDK documentation says it requires

Microsoft provides an official Gradle script that **automatically finds all dependencies**, copies them to a folder, and lists the Maven coordinates you need to map to NuGet packages.

Add one line to your Android library module's `build.gradle`:

```groovy
plugins {
    alias(libs.plugins.android.library)
}

// Add this single line — it pulls in Microsoft's dependency info script
apply from: 'https://raw.githubusercontent.com/xamarin/XamarinComponents/main/Util/AndroidGradleDependencyInfo.gradle'

android {
    // ... your existing config
}

dependencies {
    implementation files('../libs/vendor-sdk.aar')
    // ... your existing dependencies
}
```

Then run the `xamarin` task:

```bash
./gradlew :{ModuleName}:xamarin
```

This does three things automatically:
1. **Builds** the release AAR (`bundleReleaseAar`)
2. **Copies** all resolved AAR/JAR dependencies to a `xamarin/` folder (renamed with group IDs for clarity)
3. **Prints** a complete list of all Maven artifacts with their group ID, artifact ID, and version

**Example output:**

```
===================================================
                    XAMARIN                        
===================================================
 Here is a list of local and maven artifact
 dependencies in your project's module.

DEPENDENCIES:  {GROUPID}:{ARTIFACTID} ({VERSION})

Maven Artifact: androidx.core:core (1.12.0)
Maven Artifact: com.google.code.gson:gson (2.10.1)
Maven Artifact: com.squareup.okhttp3:okhttp (4.12.0)
Maven Artifact: com.squareup.retrofit2:retrofit (2.9.0)
Maven Artifact: org.jetbrains.kotlin:kotlin-stdlib (1.9.22)
---------------------------------------------------
Library Output: build/outputs/aar/VendorModule-release.aar
                (This is probably what you want to bind)
===================================================
```

> **📦 Source:** [xamarin/XamarinComponents — AndroidGradleDependencyInfo.gradle](https://github.com/xamarin/XamarinComponents/blob/main/Util/AndroidGradleDependencyInfo.gradle)


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

**If a NuGet package exists** for the dependency, always prefer it — add it as a `<PackageReference>` in the binding project.

**If no NuGet package exists**, you can use the AAR/JAR files directly from the `xamarin/` folder. However, there is a critical gotcha:

> [!CAUTION]
> **Do NOT add dependency AAR/JAR files to the binding project.** If you include a dependency AAR in your binding project alongside the vendor SDK AAR, the binding generator will try to generate C# bindings for **both** libraries. This creates a massive Metadata.xml headache — you'll be fixing hundreds of binding errors for libraries you don't even need to call from C#.

**The correct approach:** Add dependency AAR/JAR files directly to the **.NET MAUI app project** (not the binding project). The binding project only needs the vendor SDK AAR — the one you actually want to bind. The dependency AARs just need to be present at build time for the linker to resolve references.

```xml
<!-- In your MAUI App .csproj (NOT the binding project) -->
<ItemGroup Condition="'$(TargetFramework)' == 'net9.0-android'">
    <!-- Reference the binding project -->
    <ProjectReference Include="../AndroidVendorBindings/AndroidVendorBindings.csproj" />
    
    <!-- Dependencies that don't have NuGet packages — include directly -->
    <AndroidAarLibrary Include="NativeLibs/logback-android-3.0.0.aar" />
    <AndroidJavaLibrary Include="NativeLibs/vendor-data-processing-2.7.jar" />
</ItemGroup>
```

```xml
<!-- In your Binding Project .csproj — ONLY the SDK you want to bind -->
<ItemGroup>
    <AndroidLibrary Include="Jars/vendor-sdk.aar" />
    <!-- Do NOT add dependency AARs here! -->
</ItemGroup>
```

> **💡 In short:** The binding project binds. The MAUI project references. Keep them separate.

Add NuGet dependencies to the binding project `.csproj`:

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

    <!-- ONLY the vendor SDK you want to bind -->
    <ItemGroup>
        <AndroidLibrary Include="Jars/vendor-sdk.aar" />
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

> [!TIP]
> **Stuck on Metadata.xml?** Check `obj/Debug/net9.0-android/api.xml` — this is the raw API definition the binding generator produces from the Java bytecode. It shows the exact package names, class names, method signatures, and XPath paths you need to target in your `Metadata.xml`. If your XPath isn't matching, compare it against what's actually in `api.xml`.

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

### 🔍 Debugging & Inspection Tools

**Inspecting AAR/JAR internals with [decompiler.com](https://www.decompiler.com/):**

If you need to deep dive into what's inside an AAR or understand the Java class structure, you can use the online decompiler:

1. Rename the `.aar` file to `.zip`
2. Extract it — you'll find a `classes.jar` inside
3. Upload `classes.jar` to [decompiler.com](https://www.decompiler.com/)
4. Browse the full Java source definitions — see class hierarchies, method signatures, and interfaces

This is invaluable when you need to understand what a method expects or when Metadata.xml errors reference obfuscated classes you can't easily identify.

**Inspecting the generated DLL with JetBrains decompiler (dotPeek / Rider):**

Once your Android binding project builds successfully, use **JetBrains dotPeek** (free standalone) or the built-in decompiler in **Rider/IntelliJ** to inspect the generated binding DLL:

1. Open the binding project's output DLL (e.g., `bin/Debug/net9.0-android/AndroidVendorBindings.dll`)
2. Browse the decompiled C# classes, methods, and namespaces
3. Verify which SDK classes were bound correctly and which are missing
4. Check that method signatures match what you expect from the Java source

This helps you confirm that your `Metadata.xml` transformations produced the correct API surface before integrating into the MAUI app.

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
