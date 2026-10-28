# Builds the OpenVPN client as a standalone executable (libovpnexec.so) for the app's built-in OpenVPN engine.
#
# Only official upstream sources are used, each under its own license:
#   openvpn/  OpenVPN 2.7.8 (github.com/OpenVPN/openvpn, tag v2.7.8)            GPL v2 + OpenSSL exception
#   mbedtls/  Mbed TLS 3.6.7 LTS release (github.com/Mbed-TLS/mbedtls)          Apache 2.0
#   lzo/      LZO 2.10 (www.oberhumer.com/opensource/lzo)                         GPL v2+
#   lz4/      LZ4 (github.com/lz4/lz4)                                             BSD 2-clause
# This file and the config.h settings below are our own; config.h is generated from OpenVPN's
# config.h.cmake.in with the same checks OpenVPN's own CMake build does for Android.
# The executable runs as its own process and the app talks to it only over OpenVPN's management socket.
cmake_minimum_required(VERSION 3.18)
project(nocward-openvpn C)
set(CMAKE_C_STANDARD 11)
add_link_options("LINKER:--build-id=none")

include(CheckIncludeFiles)
include(CheckSymbolExists)
include(CheckStructHasMember)
include(CheckTypeSize)

# ---------------------------------------------------------------- LZO 2.10
file(GLOB LZO_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/lzo/src/*.c)
add_library(lzo2 STATIC ${LZO_SOURCES})
target_include_directories(lzo2 PUBLIC lzo/include lzo/include/lzo PRIVATE lzo/src)
# Clang miscompiles parts of LZO for 32-bit ARM at higher optimisation levels
if (ANDROID_ABI STREQUAL "armeabi-v7a")
    target_compile_options(lzo2 PRIVATE -O0)
endif ()

# ---------------------------------------------------------------- LZ4 (block API only)
add_library(lz4 STATIC lz4/lib/lz4.c)
target_include_directories(lz4 PUBLIC lz4/lib)

# ---------------------------------------------------------------- Mbed TLS 3.6 LTS, with its own CMake
set(ENABLE_PROGRAMS OFF CACHE BOOL "" FORCE)
set(ENABLE_TESTING OFF CACHE BOOL "" FORCE)
set(USE_SHARED_MBEDTLS_LIBRARY OFF CACHE BOOL "" FORCE)
set(USE_STATIC_MBEDTLS_LIBRARY ON CACHE BOOL "" FORCE)
set(GEN_FILES OFF CACHE BOOL "" FORCE)            # the release tarball already has the generated files
set(MBEDTLS_FATAL_WARNINGS OFF CACHE BOOL "" FORCE)
set(INSTALL_MBEDTLS_HEADERS OFF CACHE BOOL "" FORCE)
add_subdirectory(mbedtls EXCLUDE_FROM_ALL)

# ---------------------------------------------------------------- OpenVPN config.h
set(OV ${CMAKE_CURRENT_SOURCE_DIR}/openvpn)
file(READ ${OV}/version.m4 VERSION_M4)
foreach (part MAJOR MINOR PATCH)
    string(REGEX MATCH "define\\(\\[PRODUCT_VERSION_${part}\\], *\\[([^]]*)\\]" _ "${VERSION_M4}")
    set(OPENVPN_VERSION_${part} "${CMAKE_MATCH_1}")
endforeach ()
set(OPENVPN_VERSION_RESOURCE "${OPENVPN_VERSION_MAJOR},${OPENVPN_VERSION_MINOR},0,0")
set(TAP_WIN_COMPONENT_ID "none")
set(TAP_WIN_MIN_MAJOR 0)
set(TAP_WIN_MIN_MINOR 0)
set(IFCONFIG_PATH "ifconfig")
set(IPROUTE_PATH "ip")
set(ROUTE_PATH "route")

# What OpenVPN's own CMake sets for Android: tun and routes come from the app through the management interface
set(TARGET_ANDROID YES)
set(ENABLE_ASYNC_PUSH YES)
set(ENABLE_SITNL YES)
set(ENABLE_PORT_SHARE YES)
set(HAVE_SA_FAMILY_T YES)
set(ENABLE_LZ4 YES)
set(ENABLE_LZO YES)
set(ENABLE_CRYPTO_MBEDTLS YES)
set(HAVE_PSA_CRYPTO_H YES)

check_include_files(unistd.h HAVE_UNISTD_H)
if (HAVE_UNISTD_H)
    foreach (f chroot chdir dup dup2 fork execve ftruncate nice setgid setuid setsid)
        string(TOUPPER ${f} F)
        check_symbol_exists(${f} unistd.h HAVE_${F})
    endforeach ()
    check_symbol_exists(daemon "unistd.h;stdlib.h" HAVE_DAEMON)
endif ()
check_include_files(grp.h HAVE_GRP_H)
check_symbol_exists(getgrnam grp.h HAVE_GETGRNAM)
check_include_files(libgen.h HAVE_LIBGEN_H)
check_symbol_exists(basename libgen.h HAVE_BASENAME)
check_symbol_exists(dirname libgen.h HAVE_DIRNAME)
check_include_files(pwd.h HAVE_PWD_H)
check_symbol_exists(getpwnam pwd.h HAVE_GETPWNAM)
check_include_files(sys/epoll.h HAVE_SYS_EPOLL_H)
check_symbol_exists(epoll_create sys/epoll.h HAVE_EPOLL_CREATE)
check_include_files(syslog.h HAVE_SYSLOG_H)
check_symbol_exists(openlog syslog.h HAVE_OPENLOG)
check_symbol_exists(syslog syslog.h HAVE_SYSLOG)
check_include_files(sys/mman.h HAVE_SYS_MMAN_H)
check_symbol_exists(mlockall sys/mman.h HAVE_MLOCKALL)
check_include_files(sys/socket.h HAVE_SYS_SOCKET_H)
check_symbol_exists(sendmsg sys/socket.h HAVE_SENDMSG)
check_symbol_exists(recvmsg sys/socket.h HAVE_RECVMSG)
check_symbol_exists(getsockname sys/socket.h HAVE_GETSOCKNAME)
check_struct_has_member("struct cmsghdr" cmsg_len sys/socket.h HAVE_CMSGHDR)
check_include_files(sys/time.h HAVE_SYS_TIME_H)
check_symbol_exists(gettimeofday sys/time.h HAVE_GETTIMEOFDAY)
check_symbol_exists(getrlimit sys/resource.h HAVE_GETRLIMIT)
check_symbol_exists(strlcpy string.h HAVE_STRLCPY)
check_symbol_exists(strsep string.h HAVE_STRSEP)
check_symbol_exists(strtok_r string.h HAVE_STRTOK_R)
check_include_files("sys/types.h;netinet/in.h" HAVE_NETINET_IN_H)
check_include_files(arpa/inet.h HAVE_ARPA_INET_H)
check_include_files(dlfcn.h HAVE_DLFCN_H)
check_include_files(fcntl.h HAVE_FCNTL_H)
check_include_files(err.h HAVE_ERR_H)
check_include_files(netdb.h HAVE_NETDB_H)
check_include_files(net/if.h HAVE_NET_IF_H)
check_include_files(poll.h HAVE_POLL_H)
check_include_files("sys/types.h;netinet/in.h;resolv.h" HAVE_RESOLV_H)
check_include_files(sys/ioctl.h HAVE_SYS_IOCTL_H)
check_include_files(sys/inotify.h HAVE_SYS_INOTIFY_H)
check_include_files("sys/types.h;sys/uio.h" HAVE_SYS_UIO_H)
check_include_files(sys/un.h HAVE_SYS_UN_H)
check_include_files(sys/wait.h HAVE_SYS_WAIT_H)
check_include_files("sys/types.h;netinet/in.h;netinet/ip.h" HAVE_NETINET_IP_H)
if (HAVE_NETINET_IP_H)
    set(CMAKE_EXTRA_INCLUDE_FILES netinet/ip.h)
    check_type_size("struct in_pktinfo" IN_PKTINFO)
    check_struct_has_member("struct in_pktinfo" ipi_spec_dst netinet/ip.h HAVE_IPI_SPEC_DST)
    check_type_size("struct msghdr" MSGHDR)
    set(CMAKE_EXTRA_INCLUDE_FILES)
endif ()

configure_file(${OV}/config.h.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)
configure_file(${OV}/include/openvpn-plugin.h.in ${CMAKE_CURRENT_BINARY_DIR}/openvpn-plugin.h)

# ---------------------------------------------------------------- OpenVPN itself
# Every source OpenVPN lists for its own build; platform-specific files compile to nothing elsewhere
file(GLOB OPENVPN_SOURCES ${OV}/src/openvpn/*.c ${OV}/src/compat/*.c)

# Named like a library so Android installs it into nativeLibraryDir, the only place an app may run files from
add_executable(libovpnexec.so ${OPENVPN_SOURCES})
target_include_directories(libovpnexec.so PRIVATE ${CMAKE_CURRENT_BINARY_DIR} ${OV} ${OV}/src/compat ${OV}/include)
target_compile_definitions(libovpnexec.so PRIVATE
        HAVE_CONFIG_H
        DEFAULT_DNS_UPDOWN="/system/bin/false"
        PLUGIN_LIBDIR="/system/lib")
target_compile_options(libovpnexec.so PRIVATE -fPIE)
target_link_libraries(libovpnexec.so PRIVATE mbedtls mbedx509 mbedcrypto lzo2 lz4 log -fPIE -pie)
