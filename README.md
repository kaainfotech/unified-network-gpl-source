# Unified Network: corresponding source of the built-in OpenVPN engine

Unified Network (Android, by Nocward Systems) ships a built-in OpenVPN client, `libovpnexec.so`. It is the
official OpenVPN client compiled as a standalone program. The app starts it as a separate process and talks to it
only through OpenVPN's management interface; it is not linked into the app.

This repository is the complete corresponding source of that program, as required by the GPL for OpenVPN and LZO.
Each release tag matches an app version: tag `v0.2.0` is the source for app version 0.2.0.

| Part | Folder | Upstream | License |
|---|---|---|---|
| OpenVPN 2.7.8 | `src/openvpn/` | github.com/OpenVPN/openvpn, tag v2.7.8 | GPL v2 with the OpenSSL linking exception |
| LZO 2.10 | `src/lzo/` | www.oberhumer.com/opensource/lzo, lzo-2.10.tar.gz | GPL v2 or later |
| Mbed TLS 3.6.7 LTS | `src/mbedtls/` | github.com/Mbed-TLS/mbedtls, release 3.6.7 | Apache 2.0 |
| LZ4 | `src/lz4/` | github.com/lz4/lz4 | BSD 2-clause |

`src/CMakeLists.txt` is the build script used by the app (Android NDK, CMake). `config.h` is generated at build
time from OpenVPN's own `config.h.cmake.in` with OpenVPN's Android settings (`TARGET_ANDROID`, async push, sitnl).
No upstream source file is modified. Each folder keeps its upstream license file (`COPYING` or `LICENSE`).

## Building

With the Android NDK and CMake 3.22+, point an Android Gradle `externalNativeBuild` at `src/CMakeLists.txt`, or
configure it directly with the NDK toolchain file, e.g.

```
cmake -S src -B build -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake \
      -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-26
cmake --build build
```

The result is `libovpnexec.so`, an executable named like a library so Android installs it with the app.

## Questions

Source requests and questions: open an issue in this repository.

"OpenVPN" is a registered trademark of OpenVPN Inc. Unified Network supports OpenVPN profiles; it is not an
OpenVPN product.
