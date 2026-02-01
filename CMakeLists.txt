cmake_minimum_required(VERSION 3.16)

project(qwt VERSION 6.2.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find Qt (Qt6 with Qt5 fallback)
include(FindQtVersion)
find_qt_version(COMPONENTS Core Gui Widgets PrintSupport Svg Concurrent)

# Qwt configuration
add_definitions(
    -DQWT_DLL
    -DQWT_MAKEDLL
)

# Collect all source files
file(GLOB QWT_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/src/qwt*.cpp"
)

# Exclude OpenGL related files
list(FILTER QWT_SOURCES EXCLUDE REGEX ".*_glcanvas\\.cpp$")
list(FILTER QWT_SOURCES EXCLUDE REGEX ".*_opengl_canvas\\.cpp$")

# Collect all header files
file(GLOB QWT_HEADERS
    "${CMAKE_CURRENT_SOURCE_DIR}/src/qwt*.h"
)

# Exclude OpenGL related header files
list(FILTER QWT_HEADERS EXCLUDE REGEX ".*_glcanvas\\.h$")
list(FILTER QWT_HEADERS EXCLUDE REGEX ".*_opengl_canvas\\.h$")

# Create the Qwt library
add_library(qwt STATIC)

target_sources(qwt PRIVATE ${QWT_SOURCES} ${QWT_HEADERS})

# Set library properties
set_target_properties(qwt PROPERTIES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
    LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib"
    RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin"
    ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib"
    AUTOMOC ON
    AUTOUIC OFF
    AUTORCC ON
)

# Link Qt libraries
target_link_libraries(qwt
    PUBLIC
        ${QT_TARGET_PREFIX}Core
        ${QT_TARGET_PREFIX}Gui
        ${QT_TARGET_PREFIX}Widgets
        ${QT_TARGET_PREFIX}PrintSupport
        ${QT_TARGET_PREFIX}Svg
        ${QT_TARGET_PREFIX}Concurrent
)

# Include directories
target_include_directories(qwt
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
        $<INSTALL_INTERFACE:include/qwt>
)

# Define QWT_NO_OPENGL to disable internal GL includes/checks
target_compile_definitions(qwt PUBLIC QWT_NO_OPENGL QWT_MOC_INCLUDE)

# Compiler optimizations for embedded system (T113-S3 with 128MB RAM)
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    target_compile_options(qwt PRIVATE
        # Size optimization for embedded targets
        $<$<CXX_COMPILER_ID:GNU>:-Os -ffunction-sections -fdata-sections>
        $<$<CXX_COMPILER_ID:Clang>:-Os -ffunction-sections -fdata-sections>
        $<$<CXX_COMPILER_ID:MSVC>:/O1 /GL>
    )
    
    # Strip unused sections (embedded optimization)
    if(NOT MSVC)
        target_link_options(qwt PRIVATE
            -Wl,--gc-sections
            -Wl,--strip-all
        )
    endif()
endif()

# Optional: Install configuration
install(TARGETS qwt
    EXPORT qwtTargets
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include/qwt
)

install(FILES ${QWT_HEADERS}
    DESTINATION include/qwt
)

install(EXPORT qwtTargets
    FILE qwtTargets.cmake
    NAMESPACE Qwt::
    DESTINATION lib/cmake/qwt
)

# Create a config file for find_package
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/qwtConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY AnyNewerVersion
)

configure_package_config_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/qwtConfig.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/qwtConfig.cmake"
    INSTALL_DESTINATION lib/cmake/qwt
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/qwtConfig.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/qwtConfigVersion.cmake"
    DESTINATION lib/cmake/qwt
)
