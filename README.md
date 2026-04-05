# libwebrtc-build

Builds [webrtc-sdk/webrtc](https://github.com/webrtc-sdk/webrtc) (LiveKit's WebRTC fork) from source with **system libc++** (`std::__1::`) for macOS arm64.

Pre-built WebRTC binaries from other projects bundle Chromium's custom libc++ (`std::__Cr::` ABI), which is incompatible with standard macOS toolchains and Qt6. The webrtc-sdk fork also includes useful patches: sane audio handling, desktop capture, frame encryption (E2EE), and simulcast with HW encoders.

## How it works

The GitHub Actions workflow:

1. Clones [webrtc-sdk/webrtc-build](https://github.com/webrtc-sdk/webrtc-build) (their build system)
2. Runs `run.py build macos_arm64` — fetches source, applies patches, builds with `use_custom_libcxx=false`
3. Runs `run.py package macos_arm64` — creates `libwebrtc.a` + headers
4. Verifies zero `__Cr` symbols in the output
5. Repackages as `.tar.xz` and publishes a GitHub release

## Usage

1. Create a **public** GitHub repo from these files (free CI: 2000 min/month, macOS at 10x = ~200 min)
2. Go to **Actions** → **Build WebRTC** → **Run workflow**
3. Use defaults for m144, or specify a different commit
4. Download the release artifact

## Defaults

| Field  | Value                                      |
| ------ | ------------------------------------------ |
| Commit | `770e5461892d7d1aaff7ff886ab1fc7c5cd77bd1` |
| Tag    | `m144.7559.03`                             |

## Build time

~60–90 minutes on `macos-15` runners. Each build uses ~600–900 of your monthly free minutes.
