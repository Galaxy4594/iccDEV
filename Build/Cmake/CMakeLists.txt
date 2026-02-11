#################################################################################
# Cmake Project Configuration | iccDEV Project
# Copyright (C) 2024-2026 The International Color Consortium.
#################################################################################

# Bumped to 3.28 to silence CMP0156/CMP0181 warnings
cmake_minimum_required(VERSION 3.28 FATAL_ERROR)

# ----------------------------------------------------------------------
# Project Setup
# ----------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON) 

project(RefIccMAX LANGUAGES C CXX)

# Versioning
set(REFICCMAX_VERSION_MAJOR 2)
set(REFICCMAX_VERSION_MINOR 3)
set(REFICCMAX_VERSION_MICRO 1)
set(REFICCMAX_VERSION_PATCH 4)
set(REFICCMAX_VERSION "${REFICCMAX_VERSION_MAJOR}.${REFICCMAX_VERSION_MINOR}.${REFICCMAX_VERSION_MICRO}.${REFICCMAX_VERSION_PATCH}")

# Get Git Commit
execute_process(
    COMMAND git rev-parse --short=7 HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_COMMIT_HASH
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_QUIET
)
if(GIT_COMMIT_HASH)
    set(REFICCMAX_VERSION_FULL "${REFICCMAX_VERSION}+${GIT_COMMIT_HASH}")
else()
    set(REFICCMAX_VERSION_FULL "${REFICCMAX_VERSION}")
endif()

# ----------------------------------------------------------------------
# Build Options
# ----------------------------------------------------------------------

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
endif()

option(ENABLE_SHARED_LIBS         "Build dynamic link libraries"                         OFF)
option(ENABLE_STATIC_LIBS         "Build static libraries"                               ON)
option(ENABLE_TOOLS               "Build tools (command-line + GUI)"                     ON)
option(ENABLE_TESTS               "Build tests"                                          ON)
option(ENABLE_ICCXML              "Build IccXML library support"                         ON)

# Feature Toggles
option(ICC_TRACE_NAN_ENABLED      "Enable tracing NaN inputs in debug builds"            OFF)
option(ICC_CLUT_DEBUG             "Enable CLUT debugging support"                        OFF)
option(ICC_ENABLE_ASSERTS         "Enable ICC_ASSERT traps and debug assertions"         OFF)
option(ICC_LOG_SAFE               "Enable ICC_LOG_SAFE_VAL bounds-checked logging"       OFF)

if(ICC_TRACE_NAN_ENABLED)
  add_compile_definitions(ICC_TRACE_NAN_ENABLED=1)
endif()
if(ICC_CLUT_DEBUG)
  add_compile_definitions(ICC_CLUT_DEBUG=1)
endif()
if(ICC_ENABLE_ASSERTS)
  add_compile_definitions(ICC_ENABLE_ASSERTS=1)
endif()
if(ICC_LOG_SAFE)
  add_compile_definitions(ICC_LOG_SAFE=1)
endif()

# ----------------------------------------------------------------------
# Platform Configuration (MSYS2 Static Logic)
# ----------------------------------------------------------------------

# 1. MSYS2 / MinGW Static Build Logic
if(WIN32 AND NOT MSVC AND NOT ENABLE_SHARED_LIBS)
    message(STATUS ">>> Configuring for MSYS2/Clang Static Build")

    # Force CMake to find .a files instead of .dll.a
    set(CMAKE_FIND_LIBRARY_SUFFIXES ".a")

    # Hardcoded transitive dependencies.
    # These are required because static libs (.a) do not contain dependency info.
    # -static: tells linker to avoid dynamic linking
    # -s: strips symbols (makes exe smaller)
    # The rest are dependencies of LibXml2, Tiff, and wxWidgets.
    set(MSYS2_LINK_FLAGS "-static -s")
    set(MSYS2_STATIC_LIBS "-lc++ -liconv -lbcrypt -lws2_32 -llzma -lzstd -lz -ljpeg -ljbig -ldeflate -lwebp -llerc -lsharpyuv")
    
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${MSYS2_LINK_FLAGS} ${MSYS2_STATIC_LIBS}")

    # Critical Macros: These ensure headers don't look for DLL imports
    add_compile_definitions(
        wxWidgets_USE_STATIC
        wxUSE_GUI=1
        wxNO_RTTI
        LIBXML_STATIC
        XML_STATIC
        PTW32_STATIC_LIB
    )
    
    set(wxWidgets_USE_STATIC ON)
endif()

# 2. General Compiler Flags
if(MSVC)
    add_compile_options(/std:c++17 /D_WINDOWS /EHsc /wd5045)
elseif(UNIX OR (WIN32 AND NOT MSVC))
    add_compile_options(-std=c++17 -D_WINDOWS -fexceptions -Wall -Wno-switch)
endif()

# ----------------------------------------------------------------------
# Dependencies
# ----------------------------------------------------------------------

set(PROJECT_ROOT_DIR "${CMAKE_SOURCE_DIR}/../..")

# LibXml2
find_package(LibXml2 REQUIRED)
# Essential for static build to find <libxml/parser.h>
include_directories(${LIBXML2_INCLUDE_DIR}) 

# JSON
find_package(nlohmann_json REQUIRED)
include_directories(${nlohmann_json_DIR})

# Images
find_package(TIFF REQUIRED)
find_package(PNG REQUIRED)
find_package(JPEG REQUIRED)

include_directories(${TIFF_INCLUDE_DIR})
include_directories(${PNG_INCLUDE_DIRS})
include_directories(${JPEG_INCLUDE_DIR})

# wxWidgets
if(ENABLE_TOOLS)
    find_package(wxWidgets COMPONENTS core base REQUIRED)
    include(${wxWidgets_USE_FILE})
endif()

# ----------------------------------------------------------------------
# Directory Structure
# ----------------------------------------------------------------------

include_directories(${PROJECT_ROOT_DIR}/IccProfLib/)
include_directories(${PROJECT_ROOT_DIR}/IccXML/IccLibXML/)

add_subdirectory(IccProfLib)

if(ENABLE_SHARED_LIBS)
    set(TARGET_LIB_ICCPROFLIB IccProfLib2 PARENT_SCOPE)
else()
    set(TARGET_LIB_ICCPROFLIB "IccProfLib2-static" CACHE INTERNAL "Resolved IccProfLib target")
endif()

if(ENABLE_ICCXML)
    if(NOT LibXml2_FOUND)
         include_directories(${CMAKE_CURRENT_SOURCE_DIR}/IccXML/libxml2)
         add_subdirectory(IccXML/libxml2)
    endif()
    
    add_subdirectory(IccXML)
    
    if(ENABLE_SHARED_LIBS)
        set(TARGET_LIB_ICCXML "IccXML2" CACHE INTERNAL "Resolved IccXML target")
    else()
        set(TARGET_LIB_ICCXML "IccXML2-static" CACHE INTERNAL "Resolved IccXML target")
    endif()
endif()

# Tools
if(ENABLE_TOOLS)
    function(add_tool_subdir DIR)
        if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${DIR}/CMakeLists.txt")
            add_subdirectory(${DIR})
        endif()
    endfunction()

    add_tool_subdir(Tools/IccFromXml)
    add_tool_subdir(Tools/IccToXml)
    add_tool_subdir(Tools/IccJpegDump)
    add_tool_subdir(Tools/IccApplyNamedCmm)
    add_tool_subdir(Tools/IccDumpProfile)
    add_tool_subdir(Tools/IccRoundTrip)
    add_tool_subdir(Tools/IccFromCube)
    add_tool_subdir(Tools/IccV5DspObsToV4Dsp)
    add_tool_subdir(Tools/IccApplyToLink)
    add_tool_subdir(Tools/IccApplySearch)
    add_tool_subdir(Tools/IccPngDump)
    add_tool_subdir(Tools/IccDEVCmm)
    add_tool_subdir(Tools/IccApplyProfiles)
    add_tool_subdir(Tools/IccSpecSepToTiff)
    add_tool_subdir(Tools/IccTiffDump)
    add_tool_subdir(Tools/wxProfileDump)

    # Manual Dependencies for Tools
    if(TARGET iccFromXml AND TARGET ${TARGET_LIB_ICCXML})
        add_dependencies(iccFromXml ${TARGET_LIB_ICCXML})
    endif()
    if(TARGET iccToXml AND TARGET ${TARGET_LIB_ICCXML})
        add_dependencies(iccToXml ${TARGET_LIB_ICCXML})
    endif()
endif()

if(ENABLE_TESTS)
    add_subdirectory(Testing)
endif()

# ----------------------------------------------------------------------
# Installation
# ----------------------------------------------------------------------

include(GNUInstallDirs)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}Uninstall.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Uninstall.cmake"
    IMMEDIATE @ONLY
)

add_custom_target(uninstall
    "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Uninstall.cmake"
)

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}Config.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    @ONLY
)

if(ENABLE_INSTALL_RIM)
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
            DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_DOWN_NAME}"
            COMPONENT dev)
endif()

message(STATUS "Configuration Complete: ${PROJECT_NAME} v${REFICCMAX_VERSION_FULL}")