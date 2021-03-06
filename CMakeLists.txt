#  :copyright: (c) 2017 Alex Huszagh.
#  :license: FreeBSD, see LICENSE.txt for more details.

# Description
# ===========
#
#   Use:
#       Move to a custom directory, ideally out of source, and
#       type `cmake $LXW_SOURCE $FLAGS`, where `LXW_SOURCE` is the
#       path to the libxlsxwriter project, and `FLAGS` are custom
#       flags to pass to the compiler.
#
#   Example:
#       For example, in the build directory, to build libxlsxwriter
#       and the unittests in debug mode, type:
#           cd cmake
#           cmake .. -DBUILD_TESTS=ON
#           make
#           make check_xlsxwriter_unit
#           make install
#
#   Flags:
#       ZLIB_ROOT
#           The ZLIB root directory can be specified either through
#           an environment variable (`export ZLIB_ROOT=/usr/include`)
#           or using a flag with CMake (`-DZLIB_ROOT:STRING=/usr/include`).
#           This sets the preferred search path for the ZLIB installation.
#
#       BUILD_STATIC
#           Build static libraries (default on). To build shared
#           libraries, pass `-DBUILD_STATIC=OFF` during configuration.
#
#       BUILD_TESTS
#           Build unittests (default off). To build the unittests,
#           pass `-DBUILD_TESTS=ON` during configuration.
#
#       BUILD_EXAMPLES
#           Build example files (default off). To build the examples,
#           pass `-DBUILD_EXAMPLES=ON` during configuration.
#
#       USE_STANDARD_TMPFILE
#           Use the standard tmpfile() function (default off). To enable
#           the standard tmpfile, pass `-DUSE_STANDARD_TMPFILE=ON`
#           during configuration. This may produce bugs while cross-
#           compiling or using MinGW/MSYS.
#
#   CMake Options:
#       CMake sets debug and release builds with the `CMAKE_BUILD_TYPE`
#       option, which can be set as a flag during configuration.
#       To build in release mode, pass `-DCMAKE_BUILD_TYPE=Release`
#       during configuration.
#
#   Generators:
#       CMake also supports custom build generators, such as MakeFiles,
#       Ninja, Visual Studio, and XCode. For example, to generate
#       a Visual Studio solution, configure with:
#           cmake .. -G "Visual Studio 14 2015 Win64"
#
#       For more information on using generators, see:
#           https://cmake.org/cmake/help/v3.0/manual/cmake-generators.7.html
#

set(CMAKE_LEGACY_CYGWIN_WIN32 1)
cmake_minimum_required(VERSION 2.8)

SET(PROJECT_NAME "xlsxwriter" CACHE STRING "Optional project and binary name")

project(${PROJECT_NAME} C)
enable_testing()

# OPTIONS
# -------

SET(ZLIB_ROOT "" CACHE STRING "Optional root for the ZLIB installation")

option(BUILD_STATIC "Build static libxlsxwriter" ON)
option(BUILD_TESTS "Build libxlsxwriter tests" OFF)
option(BUILD_EXAMPLES "Build libxlsxwriter examples" OFF)
option(USE_STANDARD_TMPFILE "Use the C standard library's tmpfile()" OFF)
option(IOAPI_NO_64 "Disable 64-bit filesystem support" OFF)
if(DEFINED ENV{${ZLIB_ROOT}})
    set(ZLIB_ROOT $ENV{ZLIB_ROOT})
endif()

if(IOAPI_NO_64)
    add_definitions(-DIOAPI_NO_64=1)
endif()

# CONFIGURATIONS
# --------------

if(USE_STANDARD_TMPFILE)
    add_definitions(-DUSE_STANDARD_TMPFILE)
endif()

if(BUILD_STATIC)
    if(UNIX)
        set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    elseif(MINGW OR MSYS)
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -static -static-libgcc -Wno-char-subscripts -Wno-long-long")
        add_definitions(-DUSE_FILE32API)
    elseif(MSVC)
        set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /MTd /O0 /Fd${CMAKE_BINARY_DIR}/${PROJECT_NAME}.pdb")
        set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /MT /Ox /Zi /Fd${CMAKE_BINARY_DIR}/${PROJECT_NAME}.pdb")
        set(CMAKE_C_FLAGS_MINSIZEREL "${CMAKE_C_FLAGS_MINSIZEREL} /MT /Zi /Fd${CMAKE_BINARY_DIR}/${PROJECT_NAME}.pdb")
        set(CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELWITHDEBINFO} /MT /Fd${CMAKE_BINARY_DIR}/${PROJECT_NAME}.pdb")
    endif()
endif()

enable_language(CXX)
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
find_package(ZLIB REQUIRED "1.0")
include_directories(${ZLIB_INCLUDE_DIRS})
message("zlib version: " ${ZLIB_VERSION})

# LIBRARY
# -------

add_definitions(-DNOCRYPT -DNOUNCRYPT)

# Fix for modified zconf.h on Gentoo.
if(${CMAKE_HOST_SYSTEM} MATCHES gentoo)
    add_definitions(-DOF=_Z_OF)
endif()

include_directories(include include/xlsxwriter)
file(GLOB LXW_SOURCES src/*.c)
list(APPEND LXW_SOURCES third_party/minizip/ioapi.c third_party/minizip/zip.c)
if(MSVC)
    list(APPEND LXW_SOURCES third_party/minizip/iowin32.c)
endif()
if (NOT ${USE_STANDARD_TMPFILE})
    include_directories(third_party/tmpfileplus)
    list(APPEND LXW_SOURCES third_party/tmpfileplus/tmpfileplus.c)
endif()

set(LXW_PROJECT_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
set(LXW_LIB_DIR "${LXW_PROJECT_DIR}/lib")

if(BUILD_STATIC)
    add_library(${PROJECT_NAME} STATIC ${LXW_SOURCES})
else()
    add_library(${PROJECT_NAME} SHARED ${LXW_SOURCES})
endif()

if(MSVC)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E rename
        ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.pdb
        $<TARGET_FILE_DIR:${PROJECT_NAME}>/${PROJECT_NAME}.pdb
    )
endif()

# TESTS
# -----

# Create test and runner.
#
# Args:
#   sources       Name of variable holding source files
#   target        Test name
#
macro(CreateTest sources target)
    set(output_name xlsxwriter_${target})
    set(dependencies ${output_name})

    add_executable(${output_name} ${${sources}})
    target_link_libraries(${output_name} ${PROJECT_NAME} ${ZLIB_LIBRARIES})
    add_test(NAME ${output_name}
        COMMAND ${output_name}
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    )
endmacro(CreateTest)


file(GLOB LXW_UTILITY_SOURCES test/unit/utility/test*.c)
file(GLOB LXW_XMLWRITER_SOURCES test/unit/xmlwriter/test*.c)
file(GLOB LXW_WORKSHEET_SOURCES test/unit/worksheet/test*.c)
file(GLOB LXW_SST_SOURCES test/unit/sst/test*.c)
file(GLOB LXW_WORKBOOK_SOURCES test/unit/workbook/test*.c)
file(GLOB LXW_APP_SOURCES test/unit/app/test*.c)
file(GLOB LXW_CONTENTTYPES_SOURCES test/unit/content_types/test*.c)
file(GLOB LXW_CORE_SOURCES test/unit/core/test*.c)
file(GLOB LXW_RELATIONSHIPS_SOURCES test/unit/relationships/test*.c)
file(GLOB LXW_FORMAT_SOURCES test/unit/format/test*.c)
file(GLOB LXW_STYLES_SOURCES test/unit/styles/test*.c)
file(GLOB LXW_DRAWING_SOURCES test/unit/drawing/test*.c)
file(GLOB LXW_CHART_SOURCES test/unit/chart/test*.c)
file(GLOB LXW_CUSTOM_SOURCES test/unit/custom/test*.c)
file(GLOB LXW_FUNCTIONAL_SOURCES test/functional/src/*.c)

set(LXW_UNIT_SOURCES
    test/unit/test_all.c
    ${LXW_UTILITY_SOURCES}
    ${LXW_XMLWRITER_SOURCES}
    ${LXW_WORKSHEET_SOURCES}
    ${LXW_SST_SOURCES}
    ${LXW_WORKBOOK_SOURCES}
    ${LXW_APP_SOURCES}
    ${LXW_CONTENTTYPES_SOURCES}
    ${LXW_CORE_SOURCES}
    ${LXW_RELATIONSHIPS_SOURCES}
    ${LXW_FORMAT_SOURCES}
    ${LXW_STYLES_SOURCES}
    ${LXW_DRAWING_SOURCES}
    ${LXW_CHART_SOURCES}
    ${LXW_CUSTOM_SOURCES}
)

if (BUILD_TESTS)
    # unit tests
    add_definitions(-DTESTING -DCOLOR_OK)
    CreateTest(LXW_UNIT_SOURCES unit)

    # functional tests
    # WARNING: currently doesn't work, since the Python tests expect
    # in-source builds
    #find_program(PYTHON python)
    #foreach(source ${LXW_FUNCTIONAL_SOURCES})
    #    get_filename_component(basename ${source} NAME_WE)
    #    add_executable(${basename} ${source})
    #    target_link_libraries(${basename} xlsxwriter ${ZLIB_LIBRARIES})
    #endforeach(source)
endif()

# EXAMPLES
# --------

file(GLOB LXW_EXAMPLE_SOURCES examples/*.c)

if(BUILD_EXAMPLES)
    foreach(source ${LXW_EXAMPLE_SOURCES})
        get_filename_component(basename ${source} NAME_WE)
        add_executable(${basename} ${source})
        target_link_libraries(${basename} ${PROJECT_NAME} ${ZLIB_LIBRARIES})
    endforeach(source)
endif()

# INSTALL
# -------

if(MSVC)
    if(CMAKE_CL_64)
        install(TARGETS ${PROJECT_NAME}
            LIBRARY DESTINATION "lib/x64/\${CMAKE_INSTALL_CONFIG_NAME}"
            ARCHIVE DESTINATION "lib/x64/\${CMAKE_INSTALL_CONFIG_NAME}"
        )
        install(FILES $<TARGET_FILE_DIR:${PROJECT_NAME}>/${PROJECT_NAME}.pdb
            DESTINATION "lib/x64/\${CMAKE_INSTALL_CONFIG_NAME}"
        )
    else()
        install(TARGETS ${PROJECT_NAME}
            LIBRARY DESTINATION "lib/Win32/\${CMAKE_INSTALL_CONFIG_NAME}"
            ARCHIVE DESTINATION "lib/Win32/\${CMAKE_INSTALL_CONFIG_NAME}"
        )
        install(FILES $<TARGET_FILE_DIR:${PROJECT_NAME}>/${PROJECT_NAME}.pdb
            DESTINATION "lib/Win32/\${CMAKE_INSTALL_CONFIG_NAME}"
        )
    endif()
else(MSVC)
    install(TARGETS ${PROJECT_NAME}
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
    )
endif(MSVC)
install(FILES include/xlsxwriter.h DESTINATION include)
install(DIRECTORY include/xlsxwriter
    DESTINATION include
    FILES_MATCHING PATTERN "*.h"
)
