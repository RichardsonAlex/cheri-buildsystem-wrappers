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



# pure MIPS
set(TOOLCHAIN_TARGET_TRIPLE cheri-unknown-freebsd)
set(TOOLCHAIN_EXTRA_LINKER_FLAGS " -static")
set(TOOLCHAIN_ABI_FLAGS "-mabi=sandbox")
configure_file(CheriBSDToolchainTemplate.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/CheriBSDToolchainPureStatic.cmake @ONLY)

set(TOOLCHAIN_TARGET_TRIPLE cheri-unknown-freebsd)
set(TOOLCHAIN_ABI_FLAGS "-mabi=sandbox")
set(TOOLCHAIN_EXTRA_LINKER_FLAGS)
configure_file(CheriBSDToolchainTemplate.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/CheriBSDToolchainPureDynamic.cmake @ONLY)

set(TOOLCHAIN_TARGET_TRIPLE cheri-unknown-freebsd)
set(TOOLCHAIN_EXTRA_LINKER_FLAGS)
set(TOOLCHAIN_ABI_FLAGS "-mabi=64")
configure_file(CheriBSDToolchainTemplate.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/CheriBSDToolchainMips.cmake @ONLY)

install(FILES 
    ${CMAKE_CURRENT_BINARY_DIR}/CheriBSDToolchainMips.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/CheriBSDToolchainPureDynamic.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/CheriBSDToolchainPureStatic.cmake
    DESTINATION ${CMAKE_INSTALL_DATADIR}/cmake/cheri-toolchains
)

feature_summary(WHAT ALL INCLUDE_QUIET_PACKAGES)
