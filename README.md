# Vorssaint Intel

A small fork/workaround of [Vorssaint](https://github.com/vorssaint/vorssaint-utils) for **Intel Macs**.

The official Homebrew cask currently requires Apple Silicon, but Vorssaint itself can be compiled and run on Intel Macs by changing the build target.

This fork does not try to reinvent the project. It just documents the workaround.

## Intel Mac

### 1. Clone the repository

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

You can also do it directly from Terminal:

```bash
sed -i '' 's/TARGET="arm64-apple-macosx14.0"/TARGET="x86_64-apple-macosx15.0"/' build.sh
```

### 3. Build

```bash
./build.sh
```

When everything works, you should see:

```text
✓ Bundle ready: build/stage/Vorssaint.app
```

The generated app will be here:

```text
build/stage/Vorssaint.app
```

Launch it with:

```bash
open build/stage/Vorssaint.app
```

That's it.

## Check the architecture

To make sure the binaries were actually compiled for Intel:

```bash
file build/Vorssaint
file build/com.vorssaint.utils.fan-control
file build/libVorssaintNowPlaying.dylib
```

You should see `x86_64` for all three.

## Requirements

* Intel Mac
* macOS 14 or newer
* Xcode Command Line Tools
* Git

## Important

This is a **community fork/workaround**, not an official Intel release of Vorssaint.

The project is based on the original Vorssaint source code. See the original repository for the upstream project, license, credits, and documentation:

https://github.com/vorssaint/vorssaint-utils

The build produced this way is locally signed/ad-hoc. You do **not** need a paid Apple Developer account just to build and run it locally.

## Why this exists

Homebrew currently blocks the official cask on Intel Macs because the cask is declared as Apple Silicon-only.

That does not prevent the source code from being compiled for `x86_64`.

So:

```text
Homebrew cask
    ↓
Apple Silicon only ❌

Source code
    ↓
x86_64 build ✅

Intel Mac
    ↓
Vorssaint works ✅
```

Humanity survives another software compatibility problem through the ancient technique of changing one line in a shell script.
