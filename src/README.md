# Built-in OpenVPN engine

`libovpnexec.so` is the official OpenVPN client compiled as a standalone executable. The app starts it as its
own process and talks to it only through OpenVPN's documented management interface (`net/OpenVpn.kt`), so it is
a separate program shipped alongside the app, not linked into it.

Only official upstream sources are used:

| Part | Where | Source | License |
|---|---|---|---|
| OpenVPN 2.7.8 | `openvpn/` (git submodule) | github.com/OpenVPN/openvpn, tag v2.7.8 | GPL v2 with the OpenSSL linking exception |
| Mbed TLS 3.6.7 LTS | `mbedtls/` | release tarball from github.com/Mbed-TLS/mbedtls, SHA-256 a7e8bcbec0e6f761b4af24f25677626b35f762f68eef79c08677a363212d11f6 | Apache 2.0 |
| LZO 2.10 | `lzo/` | www.oberhumer.com/opensource/lzo/download/lzo-2.10.tar.gz, SHA-256 c0f892943208266f9b6543b3ae308fab6284c5c90e627931446fb49b4221a072 | GPL v2 or later |
| LZ4 | `lz4/` (git submodule) | github.com/lz4/lz4 | BSD 2-clause |

`CMakeLists.txt` here is our own. `config.h` is generated at build time from OpenVPN's own `config.h.cmake.in`,
with the same Android settings OpenVPN's CMake build uses (`TARGET_ANDROID`, async push, sitnl).

Clone with `git clone --recurse-submodules` (or run `git submodule update --init --depth 1`) before building.

When the app is distributed, the GPL asks for the corresponding source of the GPL parts (OpenVPN and LZO) to be
offered with it: this directory, with its submodules at the pinned versions, is that source. "OpenVPN" is a
registered trademark of OpenVPN Inc.; the app only says it supports OpenVPN profiles and is not an OpenVPN product.
