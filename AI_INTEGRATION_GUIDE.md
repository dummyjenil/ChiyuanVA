# ChiyuanVA (Bcore) Virtualization Engine - AI Agent Integration Guide

> **Target Audience:** AI Coding Agents, LLMs, and Developers building Android Dual-Space, App Cloners, Sandboxes, or Virtual Spaces using `Bcore-release.aar` & `black-reflection.jar`.

---

## 1. System Overview
**ChiyuanVA (Bcore)** is a high-performance Android userspace virtualization engine (based on the BlackBox architecture) that allows cloning and executing unmodified Android APKs in an isolated sandbox without requiring root permissions.

- **Primary Binary Artifacts:**
  - `Bcore-release.aar`: Core virtual engine runtime containing native JNI libraries (`.so`), fake framework services (`BActivityManager`, `BPackageManager`, `BUserManager`, `BJobManager`, `BStorageManager`), and hook engines.
  - `black-reflection.jar`: Bytecode & reflection layer for hidden Android internal API access.
- **Compatibility:** Android 5.0 to Android 15 (API 21 - 35), ARM64-v8a, ARMeabi-v7a.

---

## 2. Dependency Setup (build.gradle)

Add the built `.aar` and `.jar` to your host Android application:

```groovy
android {
    defaultConfig {
        minSdk 21
        targetSdk 34 // or 35
        
        ndk {
            abiFilters "arm64-v8a", "armeabi-v7a"
        }
    }
    
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_21
        targetCompatibility JavaVersion.VERSION_21
    }
}

dependencies {
    // 1. Core Virtual Engine Artifacts
    implementation fileTree(dir: "libs", include: ["*.jar", "*.aar"])

    // 2. Required Supporting Dependencies
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.9.0'
    implementation 'com.github.tiann:FreeReflection:3.2.2'
    implementation 'com.moandjiezana.toml:toml4j:0.7.2'
    implementation 'androidx.work:work-runtime:2.7.1'
}
```

---

## 3. Host Manifest Requirements (`AndroidManifest.xml`)

Virtual apps need permission querying and large storage access:

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- Allow querying installed packages for cloning -->
    <uses-permission android:name="android.permission.QUERY_ALL_PACKAGES" tools:ignore="QueryAllPackagesPermission" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" android:maxSdkVersion="29" />
    <uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE" tools:ignore="ScopedStorage" />

    <application
        android:name=".app.MyApplication"
        android:allowBackup="false"
        android:supportsRtl="true"
        android:theme="@style/Theme.AppCompat.Light.NoActionBar">
        
        <!-- Main UI Activity -->
        <activity
            android:name=".ui.MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

---

## 4. Initialization Lifecycle (Application Class)

The virtual engine **must** be initialized in `attachBaseContext()` and started in `onCreate()`.

### Kotlin Implementation:
```kotlin
package com.example.virtualapp.app

import android.app.Application
import android.content.Context
import com.chiyuan.va.ChiyuanVACore
import com.chiyuan.va.app.configuration.ClientConfiguration
import com.chiyuan.va.app.configuration.AppLifecycleCallback

class MyApplication : Application() {

    override fun attachBaseContext(base: Context) {
        super.attachBaseContext(base)
        
        // 1. Core initialization hooks
        ChiyuanVACore.get().closeCodeInit()
        ChiyuanVACore.get().onBeforeMainApplicationAttach(this, base)

        // 2. Attach Base Context with Client Configuration
        ChiyuanVACore.get().doAttachBaseContext(base, object : ClientConfiguration() {
            override fun getHostPackageName(): String = packageName
            override fun isHideRoot(): Boolean = true
            override fun isHideXposed(): Boolean = true
            override fun isEnableDaemonService(): Boolean = false
            override fun isDisableFlagSecure(): Boolean = true // Allows screenshots in virtual apps
            override fun isUseVpnNetwork(): Boolean = false
        })

        // 3. Register Virtual App Lifecycle Listener
        ChiyuanVACore.get().addAppLifecycleCallback(object : AppLifecycleCallback() {
            override fun beforeCreateApplication(packageName: String?, processName: String?, context: Context?, userId: Int) {
                // Hook before virtual app's Application is instantiated
            }

            override fun afterApplicationOnCreate(packageName: String?, processName: String?, application: Application?, userId: Int) {
                // Hook after virtual app's Application.onCreate() executes
            }
        })

        ChiyuanVACore.get().onAfterMainApplicationAttach(this, base)
    }

    override fun onCreate() {
        super.onCreate()
        // 4. Start Core Virtual Services
        ChiyuanVACore.get().doCreate()
    }
}
```

---

## 5. Core API Cheat Sheet for Virtual App Development

### 5.1 Installing / Cloning Applications
```kotlin
val userId = 0 // User space ID (0 for default user, 1, 2... for multiple cloned accounts)

// Option A: Clone an app already installed on host device
val installResult = ChiyuanVACore.get().installPackageAsUser("com.whatsapp", userId)
if (installResult.isSuccess) {
    // Cloned successfully
} else {
    val errorMsg = installResult.msg
}

// Option B: Install directly from an APK File on disk
val apkFile = File("/sdcard/Download/my_app.apk")
val installResult = ChiyuanVACore.get().installPackageAsUser(apkFile, userId)
```

### 5.2 Launching Virtual Applications
```kotlin
// Launch target app in virtual environment
val success = ChiyuanVACore.get().launchApk("com.whatsapp", userId)
```

### 5.3 Querying Installed Apps
```kotlin
// Get list of all installed apps in virtual container for user 0
val installedApps = ChiyuanVACore.get().getInstalledApps(userId)
for (app in installedApps) {
    val packageName = app.packageName
    val appName = app.name
    val icon = app.icon
}
```

### 5.4 Uninstalling & Stopping Apps
```kotlin
// Force-stop a running virtual app process
ChiyuanVACore.get().stopPackage("com.whatsapp", userId)

// Completely uninstall package from virtual user space
ChiyuanVACore.get().uninstallPackageAsUser("com.whatsapp", userId)
```

### 5.5 Multi-User / Dual Space Management
```kotlin
// Create a secondary user space (e.g. for dual accounts)
val newUser = ChiyuanVACore.get().createUser(1)

// List all virtual users
val userList = ChiyuanVACore.get().users

// Delete a user space
ChiyuanVACore.get().deleteUser(1)
```

### 5.6 Fake Location & Device Spoofing
```kotlin
// Spoof GPS Location for a specific virtual app
ChiyuanVACore.get().setFakeLocation(
    packageName = "com.whatsapp",
    userId = 0,
    latitude = 37.7749,
    longitude = -122.4194
)

// Spoof Device ID / Android ID / Brand
val deviceConfig = ChiyuanVACore.get().getDeviceConfig(userId)
// deviceConfig properties can be customized per virtual user
```

### 5.7 Xposed Module Support
```kotlin
// Install Xposed Module inside virtual container
ChiyuanVACore.get().installXPModule(File("/sdcard/Download/module.apk"))

// Enable/Disable Xposed Module
ChiyuanVACore.get().setXPModuleEnable("com.example.xposedmodule", true)
```

---

## 6. Common Troubleshooting for AI Agents

1. **Class Not Found / JNI Linkage Errors:**
   Ensure `ndk { abiFilters "arm64-v8a", "armeabi-v7a" }` is defined in host app's `build.gradle` so `.so` native libraries are packed into the APK.
2. **App Crash on Android 14 / 15:**
   Ensure `ChiyuanVACore.get().closeCodeInit()` and `onBeforeMainApplicationAttach()` are invoked in `attachBaseContext()` before `doAttachBaseContext()`.
3. **Multi-process Architecture:**
   BlackBox runs virtual processes as `:p0`, `:p1`, `:daemon`, etc. Ensure singletons in host application handle multi-process initialization properly.
