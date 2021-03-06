cmake_minimum_required(VERSION 3.1)
project(rizz)

if (ANDROID OR IOS)
    set(BUNDLE ON CACHE INTERNAL "Bundle mode" FORCE)
    set(BUILD_TOOLS OFF CACHE INTERNAL "Build tools" FORCE)
    set(BUILD_EXAMPLES OFF CACHE INTERNAL "Build examples" FORCE)
    set(ENABLE_HOT_LOADING OFF CACHE INTERNAL "Enable hot-loading" FORCE)
else()
    option(BUNDLE "Bundle all plugins and game together and make a single binary" OFF)
    option(BUILD_TOOLS "Build tools and editors" ON)
    option(BUILD_EXAMPLES "Build example projects" ON)
    option(ENABLE_HOT_LOADING "Enable hot-loading of assets and plugins" ON)
    option(MSVC_COMPILE_SUMMARY "Enable compilation metrics for MSVC compiler" OFF)
endif()

if (${CMAKE_BUILD_TYPE} MATCHES "Release")
    option(ENABLE_PROFILER "Enable profiler" OFF)
else ()
    option(ENABLE_PROFILER "Enable profiler" ON)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include(glslcc)
include(bundle)

function(rizz__remove_compile_options dst_var compile_flags flags)
    separate_arguments(flags)
    foreach(flag ${flags})
        string(REPLACE "${flag}" "" compile_flags "${compile_flags}")
    endforeach()
    set(${dst_var} ${compile_flags} PARENT_SCOPE)
endfunction()

function(rizz__add_example proj_name)
    string(SUBSTRING ${proj_name} 3 -1 source_file)
    if (BUNDLE) 
        add_executable(${proj_name} ${EXEC_TYPE} examples/${proj_name}/${source_file}.c)
    else()
        add_library(${proj_name} SHARED examples/${proj_name}/${source_file}.c)
    endif()
    target_compile_definitions(${proj_name} PRIVATE COMPILE_DEFINITIONS -DEXAMPLES_ROOT="${CMAKE_CURRENT_SOURCE_DIR}/examples")
    target_link_libraries(${proj_name} PRIVATE sx)
    add_dependencies(${proj_name} rizz)
endfunction()

if (NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/bin)
endif()
if (NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/bin)
endif()

add_definitions(-D__STDC_LIMIT_MACROS)
add_definitions(-D__STDC_FORMAT_MACROS)
add_definitions(-D__STDC_CONSTANT_MACROS)
if(MSVC)
    add_definitions(-D_ITERATOR_DEBUG_LEVEL=0)
    add_definitions(-D_HAS_EXCEPTIONS=0)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS=0)
endif()
if (RPI)
    add_definitions(-D__RPI__)
endif()

if (MSVC)
    rizz__remove_compile_options(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}" "/EHsc /GR")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /GR-")
    if (MSVC_COMPILE_SUMMARY)
        add_compile_options(/d2cgsummary)
    endif()
else()
    add_compile_options(-Wno-deprecated-declarations -fPIC)

    add_compile_options("$<$<CONFIG:Debug>:-D_DEBUG>")
    add_compile_options("$<$<CONFIG:Release>:-DNDEBUG>")

    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=gnu11")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -fno-rtti -fno-exceptions")
    rizz__remove_compile_options(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}" "-fexceptions -frtti")
endif()

get_directory_property(has_parent PARENT_DIRECTORY)

# PLUGIN_TYPE, EXEC_TYPE definitions
# PLUGIN_TYPE: cmake plugin builds should include PLUGIN_TYPE: add_library(plugin ${PLUGIN_TYPE} ...)
# EXEC_TYPE: cmake executable builds should include EXEC_TYPE: add_executable(binary ${EXEC_TYPE} ...)
if (has_parent)
    if (BUNDLE)
        set(PLUGIN_TYPE STATIC PARENT_SCOPE)
    else()
        set(PLUGIN_TYPE SHARED PARENT_SCOPE)
    endif()

    if (WIN32)
        set(EXEC_TYPE WIN32 PARENT_SCOPE)
    endif()
else()
    if (BUNDLE)
        set(PLUGIN_TYPE STATIC)
    else()
        set(PLUGIN_TYPE SHARED)
    endif()

    if (WIN32)
        set(EXEC_TYPE WIN32)
    endif()
endif()

# include directory is for all projects
include_directories(include)

# native projects
set(native_projects sx 
                    rizz
                    imgui
                    sprite)

# 3rdparty projects
set(3rdparty_projects utf 
                      lz4
                      cimgui
                      imguizmo
                      remotery
                      efsw)

if (NOT ENABLE_HOT_LOADING)
    list(REMOVE_ITEM 3rdparty_projects efsw)
endif()

# examples
set(example_projects 01-hello 
                     02-quad 
                     03-drawsprite 
                     04-animsprite)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

foreach (3rdparty_project ${3rdparty_projects}) 
    add_subdirectory(3rdparty/${3rdparty_project})
    set_target_properties(${3rdparty_project} PROPERTIES FOLDER 3rdparty)
endforeach()

foreach (native_project ${native_projects})
    add_subdirectory(src/${native_project})
    if (BUNDLE)
        target_compile_definitions(${native_project} PRIVATE -DRIZZ_BUNDLE)
    endif()

    if (MSVC)
        target_compile_options(${native_project} PRIVATE /WX)
    else()
        target_compile_options(${native_project} PRIVATE 
            -Wextra 
            -Wshadow 
            -Wunreachable-code 
            -Wstrict-aliasing 
            -Werror 
            -Wno-missing-field-initializers)
        if (CMAKE_C_COMPILER_ID MATCHES "GNU")
            target_compile_options(${native_project} PRIVATE -Wno-format-truncation)
        endif()
    endif()
endforeach()

if (BUILD_EXAMPLES AND NOT BUNDLE) 
    foreach (example_project ${example_projects})
        rizz__add_example(${example_project})
        set_target_properties(${example_project} PROPERTIES FOLDER examples)
    endforeach()
endif()

# Use this command to assign executable in BUNDLE mode
# rizz_set_executable(04-animsprite PLUGINS sprite imgui NAME animsprite)
