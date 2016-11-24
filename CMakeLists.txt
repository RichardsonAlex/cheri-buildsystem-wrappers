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
feature_summary(WHAT ALL INCLUDE_QUIET_PACKAGES)
