cmake_minimum_required(VERSION 3.15 FATAL_ERROR)

# Define Project ###############################################################
project("NTOU GLFW Beginner Template"
    VERSION 0.2.0
    DESCRIPTION "A quick OpenGL and GLFW setup for beginners."
)

# Create Binary Targets ########################################################

## The executable
add_executable(glfw-example)

## Static libraries
## These libraries will be consumed by our executable at the end of build.
add_library(imgui STATIC)
add_library(glad STATIC)


# Compiler Settings ############################################################
## Require C++17
target_compile_features(glfw-example PRIVATE cxx_std_17)

### Turn off compiler extensions for cross-platform portability.
set_target_properties(glfw-example imgui glad PROPERTIES
    C_EXTENSIONS OFF
    CXX_EXTENSIONS OFF
)

## Compiler flags for different compilers and different build configurations.
if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    list(APPEND CompileOptions -Wall -Wextra -pedantic-errors)
    list(APPEND CompileOptions_Debug -ggdb)
    list(APPEND CompileOptions_Release)
    target_compile_definitions(glfw-example PRIVATE $<$<CONFIG:Debug>:_GLIBCXX_DEBUG>)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    list(APPEND CompileOptions -Wall -Wextra -pedantic-errors)
    list(APPEND CompileOptions_Debug)
    list(APPEND CompileOptions_Release)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    list(APPEND CompileOptions /W4)
    list(APPEND CompileOptions_Debug)
    list(APPEND CompileOptions_Release)
endif()

## Set the compile options of targets according to the "build configuration".
#[[
    $<CONFIG:cfgs>
        1 if config is any one of the entries in comma-separated list cfgs.

    $<CONFIG:Debug>
        Evaluates to 1 if the build configuration is "Debug", or else it evaluates to 0.

    $<CONFIG:Release>
        Evaluates to 1 if the build configuration is "Release", or else it evaluates to 0.

    $<$<CONFIG:Debug>:"Debug Mode">
        Expands to "Debug Mode" when the "Debug" configuration is used, or else expands to an empty string.

    $<$<CONFIG:Release>:"Release Mode">
        Expands to "Release Mode" when the "Release" configuration is used, or else expands to an empty string.

    See details about generator-expressions at
        https://cmake.org/cmake/help/latest/manual/cmake-generator-expressions.7.html
]]
target_compile_options(glfw-example PRIVATE ${CompileOptions})
target_compile_options(glfw-example PRIVATE $<$<CONFIG:Debug>:${CompileOptions_Debug}>)
target_compile_options(glfw-example PRIVATE $<$<CONFIG:Release>:${CompileOptions_Release}>)


# Build Information ############################################################
message(STATUS)
message(STATUS "Build Information")
message(STATUS "\tHost System Name:       ${CMAKE_HOST_SYSTEM_NAME}" )
message(STATUS "\tHost System Version:    ${CMAKE_HOST_SYSTEM_VERSION}" )
message(STATUS "\tHost System Processor:  ${CMAKE_HOST_SYSTEM_PROCESSOR}" )
message(STATUS)
message(STATUS "\tC Compiler:             ${CMAKE_C_COMPILER_ID} ${CMAKE_C_COMPILER_VERSION}")
message(STATUS "\tC Compiler Path:        ${CMAKE_C_COMPILER}")
message(STATUS)
message(STATUS "\tC++ Compiler:           ${CMAKE_CXX_COMPILER_ID} ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "\tC++ Compiler Path:      ${CMAKE_CXX_COMPILER}")
message(STATUS)

if(MSVC)
  message(STATUS "\tMSVC Version: ${MSVC_VERSION}")
  message(STATUS "\tMSVC Toolset: ${MSVC_TOOLSET_VERSION}")
endif()

if(CMAKE_GENERATOR MATCHES "Xcode")
  message( STATUS "\tXcode Version: ${XCODE_VERSION}" )
endif()

if(CMAKE_SYSTEM_NAME MATCHES "Darwin")
  message(STATUS "\tMacOS SDK: ${CMAKE_OSX_SYSROOT}")
endif()


# Find External Libraries ######################################################
find_package(OpenGL REQUIRED)
find_package(glfw3 REQUIRED)
# find_package(glm REQUIRED)


# Include Path ################################################################
target_include_directories(imgui PUBLIC "ext/imgui")
target_include_directories(glad PUBLIC "ext/glad/include")
target_include_directories(glfw-example PRIVATE
    "ext/imgui/backends"
)


# Source Files #################################################################
## Collect source files and assign them to variables.
file(GLOB GLAD_Source "ext/glad/src/glad.c")
file(GLOB ImGui_Source CONFIGURE_DEPENDS
    "ext/imgui/*.cpp"
)
file(GLOB ImGui_Impl_Source CONFIGURE_DEPENDS
    "ext/imgui/backends/imgui_impl_opengl3.cpp"
    "ext/imgui/backends/imgui_impl_glfw.cpp"
)
file(GLOB_RECURSE My_Source CONFIGURE_DEPENDS
    "src/*.cpp"
)

## Assign source files to the binary targets (glfw-example, glad, and imgui).
target_sources(glad PRIVATE ${GLAD_Source})
target_sources(imgui PRIVATE ${ImGui_Source})
target_sources(glfw-example PRIVATE
    ${My_Source}
    ${ImGui_Impl_Source} # Compile them with your source because they need GLFW headers
)

# Link the Libraries ###########################################################
target_link_libraries(glad PRIVATE ${CMAKE_DL_LIBS})
target_link_libraries(glfw-example PRIVATE
    glad
    imgui
    glfw
    # glm::glm
    OpenGL::GL
)

if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9)
        target_link_libraries(glfw-example PRIVATE stdc++fs) # C++ filesystem
    endif()
endif()

# Create Symlink to Project Resources ###########################################
#[[
    Create a symbolic link from build tree to the shaders in source tree, so the
    executable in build tree will be able to find them.

    Tell CMake to create a custom target that performs the symlink action, and let
    our glfw-example depends on it.

    NOTICE: Enable Developer Mode on MS Windows to create symbolic links.
]]

add_custom_target(res-symlink
    COMMAND ${CMAKE_COMMAND} -E create_symlink
        "${CMAKE_SOURCE_DIR}/shaders"
        "${CMAKE_BINARY_DIR}/shaders"
    DEPENDS
        "${CMAKE_SOURCE_DIR}/shaders" # Make sure the directory exists
    COMMENT "Creating symlink from build tree to project resources..."
    VERBATIM
)

## Always create the symlink before building glfw-example.
add_dependencies(glfw-example res-symlink)

# CPack ########################################################################
#[[
    Package source files of the project with CPack module.

    Command:
        cmake --build <build-dir> --target package_source

    You'll see a ZIP file named "<project-name>-<version>-Source.zip" in your build directory.
]]
include(CMakeCPack.cmake)
