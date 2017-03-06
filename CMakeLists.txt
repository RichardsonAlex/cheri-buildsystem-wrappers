#
# Copyright (c) 2016 Alex Richardson
# All rights reserved.
#
# This software was developed by SRI International and the University of
# Cambridge Computer Laboratory under DARPA/AFRL contract FA8750-10-C-0237
# ("CTSRD"), as part of the DARPA CRASH research programme.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions
# are met:
# 1. Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
# 2. Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#
# THIS SOFTWARE IS PROVIDED BY THE AUTHOR AND CONTRIBUTORS ``AS IS'' AND
# ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED.  IN NO EVENT SHALL THE AUTHOR OR CONTRIBUTORS BE LIABLE
# FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS
# OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION)
# HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
# LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
# OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF
# SUCH DAMAGE.
#
cmake_minimum_required(VERSION 3.6)
project(CheriToolchains NONE)

include(GNUInstallDirs)
include(FeatureSummary)

find_package(CheriSDK CONFIG QUIET)
set_package_properties(CheriSDK PROPERTIES
    TYPE RECOMMENDED
    URL https://github.com/CTSRD-CHERI/cheribuild
    DESCRIPTION "A CMake config file defines the CHERI SDK paths"
    PURPOSE "Used to autodetect CHERI_SDK_BINDIR and CHERI_SDK_BINDIR from and installed CHERI SDK"
)
if(CheriSDK_FOUND)
    message(STATUS "Using paths from CheriSDKConfig.cmake")
    if(NOT CHERI_SDK_BINDIR)
        set(CHERI_SDK_BINDIR ${CheriSDK_TOOLCHAIN_DIR})
    endif()
    if(NOT CHERIBSD_SYSROOT)
        set(CHERIBSD_SYSROOT ${CheriSDK_SYSROOT_DIR})
    endif()
endif()

if(CHERI_SDK_BINDIR)
    # Always update the cache value if passed on command line
    set(TOOLCHAIN_SDK_BINDIR "${CHERI_SDK_BINDIR}" CACHE STRING "Path to the CHERI SDK" FORCE)
else()
    set(TOOLCHAIN_SDK_BINDIR "${CHERI_SDK_BINDIR}" CACHE STRING "Path to the CHERI SDK")
endif()

if(CHERI_SDK_BINDIR)
    # Always update the cache value if passed on command line
    set(TOOLCHAIN_SYSROOT "${CHERIBSD_SYSROOT}" CACHE STRING "Path to the CHERIBSD sysroot" FORCE)
else()
    set(TOOLCHAIN_SYSROOT "${CHERIBSD_SYSROOT}" CACHE STRING "Path to the CHERIBSD sysroot")
endif()


if(NOT TOOLCHAIN_SDK_BINDIR)
    message(FATAL_ERROR "Could not autodetect SDK binary dir. Pass -DCHERI_SDK_BINDIR=...")
endif()
if(NOT TOOLCHAIN_SYSROOT)
    message(FATAL_ERROR "Could not autodetect CHERIBSD sysroot. Pass -DCHERIBSD_SYSROOT=...")
endif()

if(NOT EXISTS "${TOOLCHAIN_SDK_BINDIR}/clang")
    message(FATAL_ERROR "Could not find clang in ${TOOLCHAIN_SDK_BINDIR} dir. CHERI_SDK_BINDIR seems to be wrong.")
endif()
if(NOT EXISTS "${TOOLCHAIN_SYSROOT}/usr/include")
    message(FATAL_ERROR "Could not find /usr/include in ${TOOLCHAIN_SYSROOT}. CHERIBSD_SYSROOT seems to be wrong.")
endif()


include(CMakeParseArguments)
function(create_cheribsd_cmake_wrapper _toolchainName _scriptName)
    cmake_parse_arguments(cccm "" "TRIPLE;ABI_FLAGS;LINKER_FLAGS" "" ${ARGN})
    set(TOOLCHAIN_TARGET_TRIPLE ${cccm_TRIPLE})
    set(TOOLCHAIN_EXTRA_LINKER_FLAGS ${cccm_LINKER_FLAGS})
    set(TOOLCHAIN_ABI_FLAGS ${cccm_ABI_FLAGS})
    configure_file(CheriBSDToolchainTemplate.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/${_toolchainName}.cmake @ONLY)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${_toolchainName}.cmake
            DESTINATION ${CMAKE_INSTALL_DATADIR}/cmake/cheri-toolchains)

    # create the wrapper script that calls CMake with -DCMAKE_TOOLCHAIN_FILE
    set(TOOLCHAIN_FILE ${CMAKE_INSTALL_FULL_DATADIR}/cmake/cheri-toolchains/${_toolchainName}.cmake)
    configure_file(cmake-for-cheribsd.sh.in ${CMAKE_CURRENT_BINARY_DIR}/${_scriptName} @ONLY)
    install(PROGRAMS ${CMAKE_CURRENT_BINARY_DIR}/${_scriptName}
            DESTINATION ${CMAKE_INSTALL_BINDIR})
endfunction()

# CheriABI
create_cheribsd_cmake_wrapper(CheriBSDToolchainCheriABIStatic cmake-for-cheribsd-cheriabi-static.sh
    TRIPLE cheri-unknown-freebsd
    ABI_FLAGS "-mabi=sandbox"
    LINKER_FLAGS " -static -Wl,-melf64btsmip_cheri_fbsd"
)
create_cheribsd_cmake_wrapper(CheriBSDToolchainCheriABIDynamic cmake-for-cheribsd-cheriabi-dynamic.sh
    TRIPLE cheri-unknown-freebsd
    ABI_FLAGS "-mabi=sandbox"
    LINKER_FLAGS " -Wl,-melf64btsmip_cheri_fbsd"
)
# CheriABI using LLD
create_cheribsd_cmake_wrapper(CheriBSDToolchainCheriABIStaticWithLLD cmake-for-cheribsd-cheriabi-static-with-lld.sh
    TRIPLE cheri-unknown-freebsd
    ABI_FLAGS "-mabi=sandbox"
    LINKER_FLAGS " -static -fuse-ld=lld -Wl,-melf64btsmip_cheri_fbsd"
)
create_cheribsd_cmake_wrapper(CheriBSDToolchainCheriABIDynamicWithLLD cmake-for-cheribsd-cheriabi-dynamic-with-lld.sh
    TRIPLE cheri-unknown-freebsd
    ABI_FLAGS "-mabi=sandbox"
    LINKER_FLAGS " -fuse-ld=lld -Wl,-melf64btsmip_cheri_fbsd"
)
# MIPS only:
create_cheribsd_cmake_wrapper(CheriBSDToolchainMIPSStatic cmake-for-cheribsd-mips-static.sh
    TRIPLE mips64-unknown-freebsd
    ABI_FLAGS "-mabi=64"
    LINKER_FLAGS " -static"
)
create_cheribsd_cmake_wrapper(CheriBSDToolchainMIPSDynamic cmake-for-cheribsd-mips-dynamic.sh
    TRIPLE mips64-unknown-freebsd
    ABI_FLAGS "-mabi=64"
    LINKER_FLAGS ""
)
create_cheribsd_cmake_wrapper(CheriBSDToolchainMIPSStaticWithLLD cmake-for-cheribsd-mips-static-with-lld.sh
    TRIPLE mips64-unknown-freebsd
    ABI_FLAGS "-mabi=64"
    LINKER_FLAGS " -fuse-ld=lld -static"
)
create_cheribsd_cmake_wrapper(CheriBSDToolchainMIPSDynamicWithLLD cmake-for-cheribsd-mips-dynamic-with-lld.sh
    TRIPLE mips64-unknown-freebsd
    ABI_FLAGS "-mabi=64"
    LINKER_FLAGS " -fuse-ld=lld"
)
feature_summary(WHAT ALL INCLUDE_QUIET_PACKAGES)
