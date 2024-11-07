cmake_minimum_required(VERSION 3.20)

project(dng
    VERSION 1.7.1
    DESCRIPTION "Adobe DNG SDK"
    LANGUAGES CXX C
)

find_library(JXL NAMES jxl REQUIRED)
find_library(JXLThreads NAMES jxl_threads REQUIRED)
find_library(JXLCms NAMES jxl_cms REQUIRED)
find_path(JXL_INCLUDE_DIRS jxl/decode.h)

# Find dependencies for static builds as libjxl doesn't provide CMake configs
# to find them automatically.
#
find_library(HWY_LIBRARY           NAMES hwy REQUIRED)
find_library(BROTLI_COMMON_LIBRARY NAMES brotlicommon brotlicommon-static REQUIRED)
find_library(BROTLI_DEC_LIBRARY    NAMES brotlidec brotlidec-static REQUIRED)
find_library(BROTLI_ENC_LIBRARY    NAMES brotlienc brotlienc-static REQUIRED)
find_library(LCMS2_LIBRARY         NAMES lcms2 REQUIRED)

set(JPEGXL_DEPENDENCIES
    ${HWY_LIBRARY}
    ${BROTLI_COMMON_LIBRARY}
    ${BROTLI_DEC_LIBRARY}
    ${BROTLI_ENC_LIBRARY}
    ${LCMS2_LIBRARY})
# For static builds libjxl requires -DJXL_STATIC_DEFINE
#
if (JPEGXL_LIBRARY MATCHES "jxl-static")
    add_definitions(-DJXL_STATIC_DEFINE)
endif()


find_package(ZLIB REQUIRED)

if (NOT TARGET XMP::XMPCore)
    find_package(XMP CONFIG REQUIRED)
endif()

message("XMP INCLUDE DIR: ${XMP_INCLUDE_DIR}")
get_property(XMP_LIB TARGET XMP::XMPCore PROPERTY LOCATION)
get_property(XMP_FILES_LIB TARGET XMP::XMPFiles PROPERTY LOCATION)

message("XMP CORE Library: ${XMP_LIB}")
message("XMP Files Library: ${XMP_FILES_LIB}")

file(GLOB DNG_SOURCES src/*.cpp)

add_library(dng ${DNG_SOURCES})
# DNG and headers require C++ 11
target_compile_features(dng PUBLIC cxx_std_11)
target_include_directories(dng PRIVATE ${JXL_INCLUDE_DIRS} ${XMP_INCLUDE_DIR})
target_link_libraries(dng PUBLIC
    ZLIB::ZLIB XMP::XMPCore XMP::XMPFiles ${JXL} ${JXLThreads} ${JXLCms}
    ${JPEGXL_DEPENDENCIES})

set(DNG_DEFINITIONS
    -DqDNGUseXMP=1
    -DqDNGXMPFiles=1
    -DqDNGXMPDocOps=0
)

if (APPLE)
    if (CMAKE_SYSTEM_NAME MATCHES "iOS")
        list(APPEND DNG_DEFINITIONS
            -DIOS_ENV=1
            -DqiPhone=1
        )
    else()
        list(APPEND DNG_DEFINITIONS
            -DMAC_ENV=1
            -DqMacOS=1
        )
    endif()
elseif (ANDROID)
    list(APPEND DNG_DEFINITIONS
        -DANDROID_ENV=1
        -DqAndroid=1
    )
elseif (WIN32)
    list(APPEND DNG_DEFINITIONS
        -DWIN_ENV=1
        -DqWinOS=1
    )
else()
    list(APPEND DNG_DEFINITIONS
        -DUNIX_ENV=1
        -DqLinux=1
    )
endif()

if (CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64" OR CMAKE_SYSTEM_PROCESSOR MATCHES "arm64")
    list(APPEND DNG_DEFINITIONS -DqDNGBigEndian=0)
endif()

target_compile_definitions(dng PUBLIC ${DNG_DEFINITIONS})

set(DNG_INCLUDE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/src)
target_include_directories(dng PUBLIC
    $<BUILD_INTERFACE:${DNG_INCLUDE_DIRS}>
    $<INSTALL_INTERFACE:include/dng>)

# Install targets
file(GLOB DNG_HEADERS src/*.h)
install(TARGETS dng EXPORT DngTargets)
install(FILES ${DNG_HEADERS} DESTINATION include/dng)
install(EXPORT DngTargets FILE DngTargets.cmake DESTINATION lib/cmake/dng)

set(DNG_INSTALL_INCLUDE_DIR include/dng)

include(CMakePackageConfigHelpers)
include(GNUInstallDirs)
# Configure the package
configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/DNGConfig.cmake
    PATH_VARS DNG_INSTALL_INCLUDE_DIR
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/DNG
)

# Generate version file
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/DNGConfigVersion.cmake
    VERSION 1.0.0  # Replace with your actual version
    COMPATIBILITY SameMajorVersion
)

# Install the config files
install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/DNGConfig.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/DNGConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/DNG
)

