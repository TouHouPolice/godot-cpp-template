# Using the same minimum as the godot-cpp project
cmake_minimum_required(VERSION 3.17)

# Silence unused variable warning when specified from toolchain
if(CMAKE_C_COMPILER)
endif()

# Now we can specify our own project which will inherit any global cmake properties or variables that have been defined.
project(godot-cpp-template
    VERSION 1.0
    DESCRIPTION "This repository serves as a quickstart template for GDExtension development with Godot 4.0+."
    HOMEPAGE_URL "https://github.com/enetheru/godot-cpp-template/tree/main"
    LANGUAGES CXX
)


set(LIBNAME "MySample" CACHE STRING "The name of the library")
set(GODOT_PROJECT_DIR "demo" CACHE STRING "The directory of a Godot project folder")

# Make sure all the dependencies are satisfied
find_package(Python3 3.4 REQUIRED)
find_program(GIT git REQUIRED)

# Ensure godot-cpp submodule has been updated
if(NOT EXISTS "${CMAKE_CURRENT_LIST_DIR}/godot-cpp/src")
    message(NOTICE "godot-cpp bindings source not found")
    message(NOTICE "initializing/updating the godot-cpp submodule...")

    # update the c++ bindings submodule to populate it with the necessary source for the library
    execute_process(
        COMMAND git submodule update --init godot-cpp
        WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}
        COMMAND_ERROR_IS_FATAL ANY
    )
endif()
add_subdirectory(godot-cpp SYSTEM)

# Add godot-cpp's module path and include the exported functions.
# This is made available for documentation generation
set(CMAKE_MODULE_PATH "${CMAKE_MODULE_PATH};${godot-cpp_SOURCE_DIR}/cmake")
include(GodotCPPModule)

# The godot-cpp target has some of useful properties attached that can be retrieved like so.
get_target_property(GODOTCPP_SUFFIX godot::cpp GODOTCPP_SUFFIX)
get_target_property(GODOTCPP_PLATFORM godot::cpp GODOTCPP_PLATFORM)

add_library(${LIBNAME} SHARED)

target_sources(${LIBNAME}
    PRIVATE
    src/register_types.cpp
    src/register_types.h
    src/example_class.cpp
    src/example_class.h
)

# Fetch a list of the xml files to use for documentation and add to our target
file(GLOB_RECURSE DOC_XML LIST_DIRECTORIES NO CONFIGURE_DEPENDS "${PROJECT_SOURCE_DIR}/doc_classes/*.xml")

# conditionally add doc data to compile output
if(DOC_XML)
    if(GODOTCPP_TARGET MATCHES "editor|template_debug")
        target_doc_sources(${LIBNAME} ${DOC_XML})
    endif()
endif()

target_link_libraries(${LIBNAME} PRIVATE godot-cpp)

# Require at least C++17 for this target
set_property(TARGET ${LIBNAME} PROPERTY CXX_STANDARD 17)

set_target_properties(${LIBNAME}
    PROPERTIES
    # The generator expression here prevents msvc from adding a Debug or Release subdir.
    RUNTIME_OUTPUT_DIRECTORY "$<1:${PROJECT_SOURCE_DIR}/bin/${GODOTCPP_PLATFORM}>"

    PREFIX ""
    OUTPUT_NAME "${LIBNAME}${GODOTCPP_SUFFIX}"
)

if(MSVC)
    set_property(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" PROPERTY VS_STARTUP_PROJECT ${LIBNAME})
    # 1. Define a variable for the Godot Executable so it can be set via command line or GUI
    set(GODOT_EXECUTABLE "godot" CACHE FILEPATH "Path to the Godot executable")
    # 2. Tell CMake to configure the Visual Studio Debugger settings
    # Note: We use file(TO_NATIVE_PATH) to ensure Windows backslashes are used
    file(TO_NATIVE_PATH "${PROJECT_SOURCE_DIR}/${GODOT_PROJECT_DIR}" GODOT_PROJECT_NATIVE_PATH)
    
    set_target_properties(${LIBNAME} PROPERTIES
        # Point the debugger to Godot
        VS_DEBUGGER_COMMAND "${GODOT_EXECUTABLE}"
        
        # Pass the path to your demo project
        # without -e to debug the scene window, with -e to debug the editor
        VS_DEBUGGER_COMMAND_ARGUMENTS "--path \"${GODOT_PROJECT_NATIVE_PATH}\" "
        
        # Set working directory to the project root
        VS_DEBUGGER_WORKING_DIRECTORY "${GODOT_PROJECT_NATIVE_PATH}"
        
        # Ensure the environment matches what Godot needs (optional)
        # VS_DEBUGGER_ENVIRONMENT "PATH=${GODOT_PROJECT_NATIVE_PATH}/bin;%PATH%"
    )
endif()

set(GODOT_PROJECT_BINARY_DIR "${PROJECT_SOURCE_DIR}/${GODOT_PROJECT_DIR}/bin/${GODOTCPP_PLATFORM}")

add_custom_command(TARGET ${LIBNAME} POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy "$<TARGET_FILE:${LIBNAME}>" "${GODOT_PROJECT_BINARY_DIR}/$<TARGET_FILE_NAME:${LIBNAME}>"
)

# NEW COMMAND (Copies the PDB - crucial for debugging)
if(MSVC)
    add_custom_command(TARGET ${LIBNAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different "$<TARGET_PDB_FILE:${LIBNAME}>" "${GODOT_PROJECT_BINARY_DIR}/$<TARGET_PDB_FILE_NAME:${LIBNAME}>"
    )
endif()
