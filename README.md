# OpenMW-VR for Meta Quest (native APK)

Self-contained build of OpenMW-VR as a standalone Quest APK (OpenXR + GLES via gl4es).
Everything needed lives in this repo:

```
android/
  buildscripts/   cross-compiles the engine + all native deps for arm64-v8a
  app/            minimal VR APK that loads libopenmw.so straight into VR
```

Based on https://gitlab.com/madsbuvi/openmw/-/tree/openmw-51 and https://github.com/xyzz/openmw-android

## Game data

The APK ships no Morrowind data. Put your Morrowind install in **either** location:

```sh
# Option A — full Morrowind folder on shared storage (contains "Data Files/"):
adb push Morrowind/ /sdcard/Morrowind/
# Option B — app-scoped folder, just the Data Files contents (no permission needed):
adb push "Morrowind/Data Files/." /sdcard/Android/data/org.openmw.vr/files/data/
```

`OpenMWActivity` regenerates `…/files/config/openmw.cfg` on each launch, scanning
`/sdcard/Morrowind/Data Files`, `/sdcard/Morrowind`, and the app-scoped `data/` folder — it adds a
`data=` line for each that exists and a `content=` entry per game file found (Morrowind/Tribunal/
Bloodmoon ordered first). For Option A, grant the storage permission when prompted, or pre-grant:

```sh
adb shell pm grant org.openmw.vr android.permission.READ_EXTERNAL_STORAGE
```

## Install & run on the headset (one command)

Once the Quest is reachable over adb (USB, or `adb connect <ip>:5555`):

```sh
android/run-on-device.sh ~/Morrowind   # pushes data, installs, grants perms, launches, tails log
android/run-on-device.sh               # if data is already on the device
```

It captures the startup log to `android/omw-device.log` (filtered to OpenMW/SDL/OpenXR/crash tags).

## Build

The Quest build was produced in Ubuntu under WSL2. It targets `arm64-v8a`, uses
the Android NDK downloaded by `build.sh`, and packages with Gradle 8.7.

### Prerequisites

Install a current Ubuntu WSL2 distribution and the Android SDK command-line tools.
`ANDROID_HOME` must point to an SDK containing Build Tools, Platform Tools, and
Android Platform 35. The build host also needs the usual native build tools:

```sh
sudo apt update
sudo apt install -y build-essential cmake ninja-build git curl unzip zip \
  autoconf automake libtool pkg-config python3 openjdk-17-jdk ccache
sdkmanager 'platform-tools' 'platforms;android-35' 'build-tools;35.0.0'
```

Set the SDK location before building (adjust it if your SDK is elsewhere):

```sh
export ANDROID_HOME="$HOME/Android/Sdk"
export PATH="$ANDROID_HOME/platform-tools:$ANDROID_HOME/cmdline-tools/latest/bin:$PATH"
```

### Build the Quest 3 APK

```sh
cd android/buildscripts
./build.sh            # downloads the NDK, builds deps + libopenmw.so, deploys into app/
cd ..
./gradlew :app:assembleDebug
cp app/build/outputs/apk/debug/app-debug.apk ../OpenMW-VR-Quest3-debug.apk
```

`build.sh` enables `ccache` by default, so subsequent native builds reuse compiled
objects. Set `CCACHE=false` only when a completely uncached compilation is needed.
The resulting APK is `OpenMW-VR-Quest3-debug.apk`; it is arm64-only and intended
for Meta Quest 3. The APK is deliberately not versioned in Git because it is about
1 GB—publish it as a GitHub Release asset instead.

* Morrowind.
* Dark Messiah of Might and Magic, for its animations, sounds and effects. Without it the game runs
  with Morrowind's animations and the features that need Dark Messiah's are turned off.
* Linux: Debian 13, Ubuntu 24.04 with updates, Fedora 45 or a newer distribution,
  x86-64 (glibc 2.38+, GCC 14-compatible C++ runtime and desktop OpenGL libraries).
* Windows: Windows 10 or newer, 64-bit.

## Debugging

`build.sh` keeps unstripped libs in `buildscripts/symbols/arm64-v8a/` for symbolizing
tombstones. Watch startup with:

```sh
adb logcat -s OpenMW SDL openxr VrApi DEBUG
```
