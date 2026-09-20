---
title: "For Google TV (Android TV OS): Launching a Minimal App with Android Studio + Kotlin"
description: "Target: Google TV (= Android TV OS) Goal: Get to the point where a minimal app can be launched using..."
pubDatetime: 2026-01-28T15:06:09.992Z
updatedDate: 2026-02-01T12:16:43.177Z
---

Target: **Google TV (= Android TV OS)**
Goal: Get to the point where a minimal app can be launched using Android Studio + Kotlin.

## Environment Used (Android Studio)

```text
Android Studio Otter 3 Feature Drop | 2025.2.3
Build #AI-252.28238.7.2523.14688667, built on January 9, 2026
Runtime version: 21.0.8+-14196175-b1038.72 amd64
VM: OpenJDK 64-Bit Server VM by JetBrains s.r.o.
Toolkit: sun.awt.windows.WToolkit
Windows 11.0
GC: G1 Young Generation, G1 Concurrent GC, G1 Old Generation
Memory: 2048M
Cores: 8
Registry:
  ide.experimental.ui=true
```

## 1. Create a New Project

1. Android Studio → **New Project**
2. Template: **Television → Empty Activity**
3. Language: **Kotlin**
4. **minSdk must be 21 or higher** (TV assumes API 21+)
5. After creation, first run **Gradle Sync** (check progress in the bottom-right status bar)

   * The first run may take quite a long time

## 2. Run (Emulator or Physical Device)

### A. TV Emulator (Incomplete)

* Android Studio → Tools → **Device Manager** → **+ Create Virtual Device**
* Add Device → **TV → Television (720p)**
* Configure virtual device → API: **API 34 (Android 14 / UpsideDownCake)**

  * Assumed to match Google TV
* Aborted after this point, so omitted

### B. Google TV Physical Device

#### 1) Check the Location of `adb`

1. Android Studio → Tools → **SDK Manager**
2. Check **Android SDK Locations**
3. Example: `{Android SDK Location}/platform-tools/adb.exe`

#### 2) Enable Developer Options on the Device

* On the device: **Settings → About → Build**
  → Tap repeatedly → **Enable Developer options**

#### 3) Debug Settings (USB / Wi-Fi)

* Developer options

  * **USB debugging**: ON if using USB
  * **Wireless debugging**: For Wi-Fi

    * Enable (it seems to work if you disable it once and then re-enable it)
    * Open **“Pair device with pairing code”**
      → A PIN, etc. is displayed; use it for pairing

#### 4) Pair / Connect with `adb`

```bash
adb kill-server
adb start-server

# Close and reopen the PIN screen on the Google TV side, then run
adb pair {IP}:{PORT} {PIN}

adb shell
# Even when pair was successful, there were many times it wouldn’t enter shell
```

#### 5) Run on the Physical Device from Android Studio

1. Android Studio → Tools → **Device Manager**
2. Confirm that the physical device has been added
3. Run **Run 'app'**
   → The app launches on the Google TV device

## References

* [Create and run a TV app | Android TV | Android Developers](https://developer.android.com/training/tv/get-started/create)
* [Use Jetpack Compose on Android TV | Android Developers](https://developer.android.com/training/tv/playback/compose)
