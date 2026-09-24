# chdman for Android

Automated GitHub Actions pipeline that cross-compiles `chdman`, the CHD (Compressed Hunks of Data) tool from the [MAME](https://github.com/mamedev/mame) project, for Android, and publishes the resulting binaries as a GitHub Release.

## What is chdman?

`chdman` is MAME's command-line utility for creating, verifying, extracting, and inspecting CHD files, the compressed disc/media image format MAME (and many other emulators and frontends) use for CD, DVD, laserdisc, and hard disk images. The official MAME project only ships prebuilt binaries for desktop platforms, so this repository builds `chdman` for Android instead, straight from the same source tag MAME itself releases.

## What this repository does

The workflow in [`.github/workflows/ci-android.yml`](.github/workflows/ci-android.yml) runs on demand and:

1. Checks out this repository together with `mamedev/mame` at a pinned tag (`MAME_SRC_VER`, currently `mame0289`).
2. Applies [`mame0289.patch`](mame0289.patch) to that MAME source tree (see below for what it changes).
3. Uses the Android NDK to build `chdman` only, no emulator core and no SDL dependency, for all four Android ABIs in parallel: `android-arm`, `android-arm64`, `android-x86`, and `android-x64`.
4. Strips each resulting binary and uploads it as a workflow artifact.
5. Once every architecture has finished, collects all four binaries and publishes them together as a single GitHub Release.

The release is named and tagged the same way mamedev/mame names its own releases: tag `mame0289`, title `MAME 0.289`, generated automatically from `MAME_SRC_VER` so it always matches the upstream MAME version the binaries were built from.

## Repository contents

| File | Purpose |
| --- | --- |
| `.github/workflows/ci-android.yml` | The build and release workflow described above |
| `mame0289.patch` | Patch applied to the `mamedev/mame` checkout before building |

## Running a build

The workflow only runs manually:

1. Go to the **Actions** tab of this repository.
2. Select **CI (Android)**.
3. Click **Run workflow**.

To build a different MAME version, edit the `MAME_SRC_VER` value near the top of `ci-android.yml` (and update or replace the patch file if it no longer applies cleanly to that tag), then run the workflow again.

## Getting the binaries

Once a run finishes, the four `chdman` binaries are available on this repository's **Releases** page, named:

- `chdman-android-arm-mame0289`
- `chdman-android-arm64-mame0289`
- `chdman-android-x86-mame0289`
- `chdman-android-x64-mame0289`

(the `mame0289` suffix tracks whatever `MAME_SRC_VER` was set to for that build). They are also available as short-lived workflow artifacts on the run itself, but the Release is the durable place to get them from.

## Using chdman on Android

Each binary is a plain ELF executable with no GUI, built with a statically linked `libstdc++` so it does not need an extra shared library alongside it. Typical usage:

```sh
adb push chdman-android-arm64-mame0289 /data/local/tmp/chdman
adb shell chmod +x /data/local/tmp/chdman
adb shell /data/local/tmp/chdman createcd -i game.cue -o game.chd
```

The same binary works the same way from a Termux shell on-device, or when invoked by another Android app that shells out to it. Pick the binary matching the device's ABI (most modern phones and tablets are `arm64`).

## About the patch

`mame0289.patch` makes a few small changes to MAME's build system so a tools-only Android build works without dragging in the full emulator:

- Lets `make android-<arch>` build just the requested `TOOLS` target instead of the full emulator when `EMULATOR=0`.
- Skips the `SDL_INSTALL_ROOT` requirement when `--with-emulator` is not set, since `chdman` does not use SDL.
- Adds a `CUSTOM_VCS_REVISION` override so the build's embedded version string can be set explicitly instead of being derived from `git describe` (the checkout above is a shallow, tag-only checkout with no full git history to describe).
- Drops a couple of emulator-only source files (`aes256cbc.cpp`, `nanosvg.cpp`) from the tools build, and neutralizes the SDL clipboard code path, since neither is needed outside the full emulator.
- Guards the Clang ARMv8 hardware-accelerated AES/SHA-256 path in `3rdparty/lzma/C/AesOpt.c` and `Sha256Opt.c` so it is skipped on 32-bit ARM, since leaving it enabled fails to build for the `android-arm` target; `android-arm64` is unaffected and keeps using the accelerated path.

## License and disclaimer

This repository only contains build automation. `chdman` itself is part of the MAME project: MAME as a whole is distributed under GPL-2.0-or-later, while the large majority of individual files (including most of the code `chdman` is built from) are BSD-3-Clause. See [mamedev/mame's license](https://github.com/mamedev/mame/blob/master/LICENSE.md) for the full details. "MAME" is a registered trademark of its owner; this project is not affiliated with or endorsed by the MAME team.
