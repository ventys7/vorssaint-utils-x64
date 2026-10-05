# Vorssaint Intel

A small community workaround for running [Vorssaint](https://github.com/vorssaint/vorssaint-utils) on **Intel Macs**.

The official Homebrew cask is Apple Silicon-only, but the Vorssaint source itself can be compiled for `x86_64`.

This repository does **not** distribute a modified version of Vorssaint. It documents the changes needed to build it locally for Intel Macs.

## Important

This is **not an official Intel release** of Vorssaint.

The project is based on the original Vorssaint source code. For the upstream project, license, credits and documentation, see:

https://github.com/vorssaint/vorssaint-utils

This workaround was tested on an Intel Mac running macOS 15.

The build does not require a paid Apple Developer account.

## Requirements

* Intel Mac
* macOS 15 or newer
* Xcode Command Line Tools
* Git

## Why this exists

Homebrew's official Vorssaint cask is declared for Apple Silicon only, so installing it on an Intel Mac fails before the app is even built.

The source code itself can still be compiled for `x86_64`.

```text
Homebrew cask
    ↓
Apple Silicon only ❌

Vorssaint source
    ↓
x86_64 build ✅

Intel Mac
    ↓
Vorssaint works ✅
```

## Build for Intel

### 1. Clone the upstream repository

```bash
git clone https://github.com/vorssaint/vorssaint-utils.git
cd vorssaint-utils
```

### 2. Change the build target

In `build.sh`, find:

```bash
TARGET="arm64-apple-macosx14.0"
```

and change it to:

```bash
TARGET="x86_64-apple-macosx15.0"
```

Or do it directly from Terminal:

```bash
sed -i '' 's/TARGET="arm64-apple-macosx14.0"/TARGET="x86_64-apple-macosx15.0"/' build.sh
```

### 3. Add Intel CPU temperature support

Vorssaint's temperature selector only knows Apple Silicon CPU sensor layouts by default.

For Intel Macs, add:

```swift
case intel
```

to `CPUTemperaturePlatform`.

Then add the Intel core sensors:

```swift
private static let intelCPUCoreKeys: Set<String> = [
    "TC1C", "TC2C", "TC3C", "TC4C",
]
```

Make the platform detector recognise Intel:

```swift
if brand.range(of: "Intel", options: .caseInsensitive) != nil {
    return .intel
}
```

Add `.intel` to the CPU core set handling and classify the Intel `TC...` keys as CPU temperature sensors.

The four core sensors used by this workaround are:

```text
TC1C
TC2C
TC3C
TC4C
```

### 4. Make `--sensors` show Intel CPU sensors

`SensorDump` also needs to include `TC...` keys.

The temperature filter should include:

```swift
|| name.hasPrefix("TC")
```

This makes:

```bash
./build/Vorssaint --sensors
```

show the Intel CPU sensors instead of silently ignoring them.

### 5. Add the Intel GPU sensor

Some Intel Macs do not expose a GPU temperature as `Tg...`.

The Mac tested for this workaround exposes the Intel GPU temperature as:

```text
TCGC
```

Vorssaint therefore needs to recognise `TCGC` as a GPU temperature sensor in:

```text
Sources/Vorssaint/Services/SystemMonitor/SystemMonitor.swift
Sources/Vorssaint/Services/FanControl/FanControlHardware.swift
Sources/Vorssaint/Support/SelfTest.swift
```

The relevant GPU selection should include:

```swift
$0.name == "TCGC"
```

or the equivalent for the closure being used.

On the tested Intel Mac, the resulting output looks like:

```text
cpu-core  TC1C  sp78  59.00
cpu-core  TC2C  sp78  60.00
cpu-core  TC3C  sp78  59.00
cpu-core  TC4C  sp78  58.00
gpu       TCGC  sp78  59.00
```

The exact values will obviously depend on what the Mac is doing at the time.

### 6. Use a stable local signing identity

The first build may fall back to ad-hoc signing.

For repeated local builds, it is better to create the signing identity included by the project:

```bash
./Tools/setup-signing.sh
```

Then remove the previous build and rebuild:

```bash
rm -rf build
./build.sh
```

This creates a local self-signed identity and gives the application a stable code-signing identity across rebuilds.

You do not need an Apple Developer membership for this.

### 7. Build

```bash
./build.sh
```

A successful build ends with:

```text
✓ Bundle ready: build/stage/Vorssaint.app
```

The application is here:

```text
build/stage/Vorssaint.app
```

You can test it directly:

```bash
open build/stage/Vorssaint.app
```

## Check the architecture

Make sure the important binaries are actually Intel:

```bash
file build/Vorssaint
file build/com.vorssaint.utils.fan-control
file build/libVorssaintNowPlaying.dylib
```

All three should report:

```text
x86_64
```

You can also verify the application bundle itself:

```bash
codesign --verify --deep --strict --verbose=2 build/stage/Vorssaint.app
```

A valid build should report that the bundle is valid on disk.

## Install it in Applications

Once the build has been tested:

```bash
rm -rf /Applications/Vorssaint.app
cp -R build/stage/Vorssaint.app /Applications/
open /Applications/Vorssaint.app
```

At this point the app is running from `/Applications` rather than from the build directory.

## Permissions

Because this is a locally built application, macOS may treat a new build as a different code identity.

This can cause previously granted TCC permissions to disappear or stop applying.

If the permissions become stuck, reset them for Vorssaint:

```bash
pkill -x Vorssaint 2>/dev/null || true
tccutil reset All com.vorssaint.utils
killall "System Settings" 2>/dev/null || true
open /Applications/Vorssaint.app
```

Then go back to:

**System Settings → Privacy & Security**

and grant the permissions Vorssaint needs.

In particular, this workaround was tested with:

* Accessibility
* Screen Recording
* the other protected permissions used by the enabled features

Using the stable local signing identity above helps avoid unnecessary permission churn between local rebuilds.

## Automation permissions

Automation is slightly different.

Vorssaint does not simply appear in the Automation list and magically get every permission.

The permission is requested when the app actually sends an Apple Event to a target application.

The current upstream code has explicit Automation targets for:

```text
Finder
Terminal
```

Other media applications can be requested dynamically when playback control is used.

So, after resetting TCC, trigger the feature that needs the permission.

For example:

* use a Finder-related feature such as Finder Cut & Paste or a Cleaner action;
* use a feature that opens or controls Terminal;
* use the playback controls with a supported media application.

macOS should then ask whether Vorssaint can control that application.

Do **not** test this by running the AppleScript from Terminal with `osascript`, because that request belongs to Terminal rather than Vorssaint.

If Automation is completely stuck, reset only the Apple Events database for Vorssaint:

```bash
tccutil reset AppleEvents com.vorssaint.utils
```

Then relaunch Vorssaint and trigger the feature again.

## Intel temperature sensors

The Intel Macs handled by this workaround may expose several CPU-related sensors.

For example:

```text
TC1C
TC2C
TC3C
TC4C
```

are used as the four CPU core readings.

Other sensors such as:

```text
TC0E
TC0F
TC0P
TCMX
TCSA
TCXC
```

are auxiliary CPU-related thermal readings.

They are useful for diagnostics and thermal management, but they are not individual CPU cores.

The GPU on the tested Intel Mac is exposed as:

```text
TCGC
```

That is why `TCGC` is classified separately as GPU.

## Check the sensors

Run:

```bash
./build/Vorssaint --sensors
```

On the tested Mac, the output includes both CPU and GPU readings:

```text
cpu-core  TC1C  sp78  ...
cpu-core  TC2C  sp78  ...
cpu-core  TC3C  sp78  ...
cpu-core  TC4C  sp78  ...
gpu       TCGC  sp78  ...
```

Battery sensors such as `TB0T`, `TB1T` and `TB2T` may also appear.

## Updates

This is a local Intel workaround, so automatic application updates are not recommended.

A future upstream release can change:

* the build target;
* the temperature sensor mapping;
* the signing process;
* the permission handling;
* the structure of the affected source files.

For that reason, it is safer to keep automatic updates disabled and rebuild manually when you actually want to move to a newer upstream version.

Before updating, check the upstream changes first rather than blindly replacing the working build.

## What this workaround changes

The Intel build currently requires changes in these areas:

```text
build.sh
    ↓
x86_64 target

TemperatureSensorSelector.swift
    ↓
Intel CPU platform + TC1C–TC4C

SelfTest.swift
    ↓
TC... diagnostic output + TCGC GPU classification

SystemMonitor.swift
    ↓
Intel GPU temperature via TCGC

FanControlHardware.swift
    ↓
Intel GPU temperature via TCGC

Tools/setup-signing.sh
    ↓
stable local signing identity
```

The upstream application itself remains the source of the build.

---

Humanity survives another software compatibility problem through the ancient technique of changing one line in a shell script.

Then another five lines.

And some TCC voodoo.
