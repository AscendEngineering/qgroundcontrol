cmake_minimum_required(VERSION 3.21.1 FATAL_ERROR)

project(VideoReceiverApp LANGUAGES C CXX)

set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug;Release;RelWithDebInfo;MinSizeRel;Coverage")

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTOUIC ON)
set(CMAKE_AUTORCC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)

include(FeatureSummary)

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" OR "${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    add_compile_options(-Wall -Wextra -Wno-address-of-packed-member)
endif()

# CMake build type
# Debug Release RelWithDebInfo MinSizeRel Coverage
if (NOT CMAKE_BUILD_TYPE)
    # default to release with debug symbols
    set(CMAKE_BUILD_TYPE RelWithDebInfo CACHE STRING "Build type" FORCE)
endif()

set(QGC_ROOT ${CMAKE_SOURCE_DIR}/..)

# Add folder where are supportive functions
list(APPEND CMAKE_MODULE_PATH ${QGC_ROOT}/cmake)

# Configure Qt6 to get necessary variables
include(Qt6QGCConfiguration)
message(STATUS "Build Type: ${CMAKE_BUILD_TYPE}")
message(STATUS "Qt version: ${QT_VERSION}")
message(STATUS "Qt spec: ${QT_MKSPEC}")

set(COMPANY "Auterion")
set(COPYRIGHT "Copyright (c) 2020 VideoReceiverApp. All rights reserved.")
set(IDENTIFIER "labs.auterion.VideoReceiverApp")

include(Git)
message(STATUS "VideoReceiverApp version: ${APP_VERSION_STR}")

#=============================================================================
# ccache
#
option(CCACHE "Use ccache if available" ON)
find_program(CCACHE_PROGRAM ccache)
if (CCACHE AND CCACHE_PROGRAM AND NOT DEFINED ENV{CCACHE_DISABLE})
    set_property(GLOBAL PROPERTY RULE_LAUNCH_COMPILE "${CCACHE_PROGRAM}")
endif()

#=============================================================================
# Compile QML
#
option(COMPILE_QML "Pre-compile QML files using the Qt Quick compiler." FALSE)
add_feature_info(COMPILE_QML COMPILE_QML "Pre-compile QML files using the Qt Quick compiler.")
if(COMPILE_QML)
    find_package(Qt6QuickCompiler)

    set_package_properties(Qt6QuickCompiler PROPERTIES
        DESCRIPTION "Pre-compile QML files using the Qt Quick compiler."
        TYPE OPTIONAL
        )
endif()

#=============================================================================
# Debug QML
#
option(DEBUG_QML "Build VideoReceiverApp with QML debugging/profiling support." OFF)
add_feature_info(DEBUG_QML DEBUG_QML "Build VideoReceiverApp with QML debugging/profiling support.")
if(DEBUG_QML)
    message(STATUS "To enable the QML debugger/profiler, run with: '-qmljsdebugger=port:1234'")
    add_definitions(-DQMLJSDEBUGGER)
    add_definitions(-DQT_DECLARATIVE_DEBUG)
    add_definitions(-DQT_QML_DEBUG)
endif()

if (LINUX)
    # set to 0 to use X11 instead of Wayland
    set(USE_WAYLAND 0)
endif()

#=============================================================================
# GStreamer
#
find_package(PkgConfig)
pkg_check_modules(GST
    gstreamer-1.0>=1.18
    gstreamer-video-1.0>=1.18
    gstreamer-gl-1.0>=1.18
    egl
    )

if (GST_FOUND)
    include_directories(
        ${GST_INCLUDE_DIRS}
        )
else ()
    message(STATUS "No GStreamer!")
endif ()

#=============================================================================
# Qt6
#
find_package(Qt6 ${QT_VERSION}
    COMPONENTS
    Bluetooth
    Charts
    Concurrent
    Core
    Location
    Multimedia
    Network
    Positioning
    Quick
    QuickWidgets
    OpenGL
    Sql
    Svg
    Test
    TextToSpeech
    Widgets
    Xml
    REQUIRED
    HINTS
    ${QT_LIBRARY_HINTS}
    )

# Sets the default flags for compilation and linking.
include(CompileOptions)

include_directories(
    ${QGC_ROOT}/src
    ${QGC_ROOT}/src/Settings
    ${CMAKE_CURRENT_BINARY_DIR}
    ${Qt6Location_PRIVATE_INCLUDE_DIRS}
    VideoReceiver
    )

add_subdirectory(${QGC_ROOT}/libs/qmlglsink qmlglsink.build)
add_subdirectory(${QGC_ROOT}/src/VideoReceiver VideoReceiver.build)

set(VIDEORECIVERAPP_SOURCES main.cpp ${QGC_ROOT}/src/QGCLoggingCategory.cc)
set(VIDEORECIVERAPP_RESOURCES qml.qrc)

if(ANDROID)
    add_library(VideoReceiverApp SHARED ${VIDEORECIVERAPP_SOURCES} ${VIDEORECIVERAPP_RESOURCES})
else()
    add_executable(VideoReceiverApp ${VIDEORECIVERAPP_SOURCES} ${VIDEORECIVERAPP_RESOURCES})
endif()

target_link_libraries(VideoReceiverApp
    PRIVATE
        VideoReceiver
        Qt6::Core
        Qt6::Multimedia
        Qt6::OpenGL
        Qt6::Quick
        Qt6::QuickWidgets
)
