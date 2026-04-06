# webrtc-build

Builds [webrtc-sdk/webrtc](https://github.com/webrtc-sdk/webrtc) (NOT `libwebrtc` wrapper) from source for macOS Apple Silicon (arm64) for use in Qt6 QML C++ applications.

Pre-built WebRTC binaries from other projects bundle Chromium's custom libc++ (`std::__Cr::` ABI), which is incompatible with standard macOS toolchains and Qt6. The webrtc-sdk fork also includes useful patches: sane audio handling, desktop capture, frame encryption (E2EE), and simulcast with HW encoders.

## How it works

The GitHub Actions workflow:

1. Clones [webrtc-sdk/webrtc-build](https://github.com/webrtc-sdk/webrtc-build) and uses their `run.py` build system
2. Fetches the selected `webrtc-sdk/webrtc` branch (or a pinned commit), applies patches, and configures GN with `use_custom_libcxx=false`
3. Creates stub modulemaps for DarwinBasic/DarwinFoundation that fix the hermetic SDK issue on GitHub-hosted macOS runners
4. Builds specific targets with `autoninja`
5. Manually creates `libwebrtc.a` from `.o` files using `/usr/bin/ar`
6. Verifies ABI compatibility (zero `__Cr` symbols)
7. Packages with `rsync` for headers, generates `WebRTCConfig.cmake` for Qt6, and publishes a GitHub release

## Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `webrtc_branch` | choice | `m144_release` | Branch of `webrtc-sdk/webrtc` to build (`m146_release`, `m144_release`, `m137_release`, `m125_release`) |
| `build_type` | choice | `release` | Build type — `release` (optimised) or `debug` (sets `is_debug=true`, `enable_dsyms=true`) |
| `commit` | string | *(empty)* | Optional commit hash to pin. Overrides `webrtc_branch` when set |
| `release_tag` | string | `m144.7559.03` | Git tag to create for the GitHub release |

## Usage

1. Create a **public** GitHub repo from these files (free CI: 2 000 min/month, macOS at 10× = ~200 min)
2. Go to **Actions** → **Build WebRTC** → **Run workflow**
3. Pick a branch (or paste a commit hash to pin), choose build type, set a release tag
4. Download the release artifact

## Qt6 CMakeLists.txt integration

Extract the tarball, then add to your project's `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.21)
project(voice-video-chat LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 20)

find_package(Qt6 REQUIRED COMPONENTS Quick Multimedia)

# Point to the extracted WebRTC package
set(WebRTC_DIR "${CMAKE_SOURCE_DIR}/third_party/webrtc-macos-arm64")
find_package(WebRTC REQUIRED PATHS ${WebRTC_DIR} NO_DEFAULT_PATH)

qt_add_executable(voicevideochat
    main.cpp
    webrtcmanager.cpp
    webrtcmanager.h
)

qt_add_qml_module(voicevideochat
    URI VoiceVideoChat
    VERSION 1.0
    QML_FILES Main.qml VideoRoom.qml
)

target_link_libraries(voicevideochat PRIVATE
    Qt6::Quick
    Qt6::Multimedia
    WebRTC::WebRTC   # brings in libwebrtc.a + all macOS frameworks automatically
)
```

The `WebRTCConfig.cmake` bundled in the package creates the `WebRTC::WebRTC` imported target and automatically links the following macOS frameworks:

| Framework | Purpose |
|---|---|
| Foundation | Base types, strings, collections |
| AVFoundation | Camera / microphone access |
| CoreAudio | Low-level audio I/O |
| CoreMedia | Media timestamps / sample buffers |
| CoreVideo | Video buffer management |
| AudioToolbox | Audio codec / format conversion |
| VideoToolbox | H.264/H.265 hardware encode & decode |
| CoreGraphics | 2-D drawing, display info |
| IOSurface | Cross-process GPU surface sharing |
| Metal | GPU compute (video processing) |
| AppKit | Window / screen enumeration |
| ScreenCaptureKit | Screen sharing capture |
| CoreFoundation | C-level CF types |
| CoreServices | UTI / file-type utilities |

## Key GN args (preserved for Qt6 compatibility)

| GN arg | Value | Why |
|---|---|---|
| `use_rtti` | `true` | Qt6 uses RTTI extensively |
| `use_custom_libcxx` | `false` | Use system libc++ to match Qt6 |
| `rtc_enable_symbol_export` | `true` | Required for linking from outside WebRTC |
| `rtc_use_h264` | `true` | H.264 support via VideoToolbox |
| `ffmpeg_branding` | `"Chrome"` | Enables additional codecs |
| `treat_warnings_as_errors` | `false` | Avoids build failures on SDK version mismatches |

## Build time

~60–90 minutes on `macos-14` (Apple Silicon M1) runners. Each build uses ~600–900 of your monthly free minutes.
