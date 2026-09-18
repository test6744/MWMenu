cmake_minimum_required(VERSION 3.20)

set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

project(YimMenu CXX ASM_MASM)

if (NOT MSVC)
  message(FATAL_ERROR "Unsupported compiler used '${CMAKE_CXX_COMPILER_ID}', YimMenu only supports MSVC.")
endif()

set(SRC_DIR "${PROJECT_SOURCE_DIR}/src")

# Git commit embed
include(cmake/git.cmake)

# Fetch modules
message("\nFetching modules")
include(cmake/asmjit.cmake)
include(cmake/minhook.cmake)
include(cmake/async-logger.cmake)
include(cmake/pugixml.cmake)
include(cmake/json.cmake)
include(cmake/cpr.cmake)
include(cmake/lua.cmake)
include(cmake/imgui.cmake)
message("\nFetching custom modules")
include(cmake/gtav-classes.cmake)


# YimMenu
message(STATUS "YimMenu")
file(GLOB_RECURSE SRC_MAIN
    "${SRC_DIR}/**.hpp"
    "${SRC_DIR}/**.h"
    "${SRC_DIR}/**.cpp"
    "${SRC_DIR}/**.cc"
    "${SRC_DIR}/**.cxx"
    "${SRC_DIR}/**.asm"
)

if (MSVC)
  add_compile_options(/bigobj)
  add_compile_options("$<$<C_COMPILER_ID:MSVC>:/utf-8>")
  add_compile_options("$<$<CXX_COMPILER_ID:MSVC>:/utf-8>")
else ()
  add_compile_options(-Wa,-mbig-obj)
endif ()

add_library(YimMenu MODULE "${SRC_MAIN}")

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(TARGET YimMenu PROPERTY CXX_STANDARD 23) # 23 Because std::format is not avalible in std:c++20 for some reason. Maybe it's because i use v142 toolset.

source_group(TREE ${SRC_DIR} PREFIX "src" FILES ${SRC_MAIN} )

target_include_directories(YimMenu PRIVATE 
    "${SRC_DIR}"
    "${json_SOURCE_DIR}/single_include"
    "${gtav_classes_SOURCE_DIR}"
    "${imgui_SOURCE_DIR}"
    "${minhook_SOURCE_DIR}/src/hde"
    "${asmjit_SOURCE_DIR}/src"
)

target_precompile_headers(YimMenu PRIVATE "${SRC_DIR}/common.hpp")
target_link_libraries(YimMenu PRIVATE pugixml minhook AsyncLogger dbghelp imgui cpr lua_static asmjit::asmjit)

# Warnings as errors
set_property(TARGET YimMenu PROPERTY COMPILE_WARNING_AS_ERROR ON)

add_compile_definitions(YimMenu 
    "_CRT_SECURE_NO_WARNINGS"
    "NOMINMAX"
    "WIN32_LEAN_AND_MEAN"
    "YIM_DEV"
)

# Optimizations
if(MSVC)
    if(CMAKE_BUILD_TYPE STREQUAL "Release" OR CMAKE_BUILD_TYPE STREQUAL "Debug")
        target_compile_options(YimMenu PRIVATE "/Zi")
        target_link_options(YimMenu PRIVATE "/DEBUG:FULL" "/MAP")
    endif()

    if(OPTIMIZE)
        target_link_options(YimMenu PRIVATE "/LTCG" "/OPT:REF,ICF" "/GUARD:NO")
        if(CMAKE_BUILD_TYPE STREQUAL "Release")
            target_compile_options(YimMenu PRIVATE "/O2" "/Oi" "/Ot" "/Oy-" "/Ob3" "/sdl-" "/GL" "/GF" "/GS-" "/Gw")
        endif()
    else()
        target_compile_definitions(YimMenu PRIVATE "YIM_DEV")
        if(CMAKE_BUILD_TYPE STREQUAL "Release")
            target_link_options(YimMenu PRIVATE "/OPT:REF,ICF")
        endif()
    endif()
endif()
