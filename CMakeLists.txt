cmake_minimum_required (VERSION 3.21)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Use modern CMake approach to set static runtime
# This ensures all targets use the same runtime library
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

# Determine build type suffix
if(CMAKE_BUILD_TYPE STREQUAL "Debug" OR CMAKE_BUILD_TYPE STREQUAL "")
    set(BUILD_SUFFIX "d")
else()
    set(BUILD_SUFFIX "")
endif()

# Add subdirectories BEFORE setting project, to ensure they inherit the runtime
add_subdirectory(ext/SPIRV-Cross)

set(SHADERC_SKIP_TESTS ON)
set(SHADERC_SKIP_INSTALL ON)
# Pass the runtime library setting to shaderc
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
add_subdirectory(ext/shaderc)

project(veldrid-spirv)

include_directories(ext/SPIRV-Cross)
include_directories(ext/SPIRV-Cross/include)
# Include paths for shaderc headers (both parent and shaderc/ subfolder)
include_directories(ext/shaderc/libshaderc/include)
include_directories(ext/shaderc/libshaderc/include/shaderc)
include_directories(ext/shaderc/libshaderc_util/include)
include_directories(ext/shaderc/third_party/glslang)
include_directories(ext/shaderc/third_party/glslang/glslang)
include_directories(ext/shaderc/third_party/glslang/glslang/Include)
include_directories(ext/shaderc/third_party/glslang/SPIRV)
include_directories(ext/shaderc/third_party/spirv-tools/include)
include_directories(ext/shaderc/third_party/spirv-headers/include)

file(GLOB_RECURSE LIBVELDRID_SPIRV_SOURCES src/libveldrid-spirv/*.cpp src/libveldrid-spirv/*.hpp)

project(veldrid-spirv)

if("${CMAKE_SYSTEM_NAME}" STREQUAL "iOS")
    set(LIBRARY_TYPE STATIC)
    message( "Configured to build a static library." )
else()
    set(LIBRARY_TYPE SHARED)
    message( "Configured to build a shared library." )
endif()

# Add library paths - use the actual build type directory
link_directories(
    ${CMAKE_CURRENT_BINARY_DIR}/ext/SPIRV-Cross/${CMAKE_BUILD_TYPE}
    ${CMAKE_CURRENT_BINARY_DIR}/ext/shaderc/libshaderc/${CMAKE_BUILD_TYPE}
    ${CMAKE_CURRENT_BINARY_DIR}/ext/shaderc/libshaderc_util/${CMAKE_BUILD_TYPE}
    ${CMAKE_CURRENT_BINARY_DIR}/ext/shaderc/third_party/glslang/glslang/${CMAKE_BUILD_TYPE}
    ${CMAKE_CURRENT_BINARY_DIR}/ext/shaderc/third_party/glslang/SPIRV/${CMAKE_BUILD_TYPE}
    ${CMAKE_CURRENT_BINARY_DIR}/ext/shaderc/third_party/glslang/glslang/OSDependent/Windows/${CMAKE_BUILD_TYPE}
    ${CMAKE_CURRENT_BINARY_DIR}/ext/shaderc/third_party/spirv-tools/source/${CMAKE_BUILD_TYPE}
    ${CMAKE_CURRENT_BINARY_DIR}/ext/shaderc/third_party/spirv-tools/source/opt/${CMAKE_BUILD_TYPE}
)

add_library(veldrid-spirv ${LIBRARY_TYPE} ${LIBVELDRID_SPIRV_SOURCES})

# Handle runtime library mismatch
if(MSVC)
    add_link_options(/NODEFAULTLIB:LIBCMT /NODEFAULTLIB:LIBCPMT)
endif()

target_link_libraries(veldrid-spirv PRIVATE
    spirv-cross-core
    spirv-cross-glsl
    spirv-cross-reflect
    spirv-cross-msl
    spirv-cross-hlsl
    shaderc_combined
)

set_target_properties(veldrid-spirv PROPERTIES PREFIX "lib")
set_target_properties(veldrid-spirv PROPERTIES DEBUG_POSTFIX "")

set_target_properties(veldrid-spirv PROPERTIES XCODE_ATTRIBUTE_ENABLE_BITCODE "NO")

if("${CMAKE_SYSTEM_NAME}" STREQUAL "iOS")
    shaderc_combine_static_lib(veldrid-spirv-combined veldrid-spirv)
endif()
