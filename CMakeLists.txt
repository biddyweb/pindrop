# Copyright 2014 Google Inc. All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
cmake_minimum_required(VERSION 2.8.12)

project(pindrop)

option(pindrop_multistream "Support multiple channels of streaming audio" off)

if(pindrop_multistream)
  add_definitions(-DPINDROP_MULTISTREAM)
endif()

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/bin)

# Temporary files (like object files) created while compiling projects.
set(tmp_dir ${CMAKE_CURRENT_BINARY_DIR}/obj)

# Directory which contains the source for 3rd party libraries.
get_filename_component(
    third_party_root "${CMAKE_CURRENT_SOURCE_DIR}/../../../../external" REALPATH)
# Directory which contains source for FPL libraries.
get_filename_component(
    fpl_root "${CMAKE_CURRENT_SOURCE_DIR}/../../libs" REALPATH)

# If the dependencies directory exists, assume this is the root directory for
# all libraries required by this project.
set(dependencies_root "${CMAKE_CURRENT_SOURCE_DIR}/dependencies")
if(EXISTS "${dependencies_root}")
  set(third_party_root "${dependencies_root}")
  set(fpl_root "${dependencies_root}")
endif()

# Configurable locations of dependencies of this project.
set(dependencies_gtest_dir "${fpl_root}/googletest"
    CACHE PATH "Directory containing the GoogleTest library.")
set(dependencies_flatbuffers_dir "${fpl_root}/flatbuffers"
    CACHE PATH "Directory containing the Flatbuffers library.")
set(dependencies_mathfu_dir "${fpl_root}/mathfu"
    CACHE PATH "Directory containing the MathFu library.")
set(dependencies_sdl_dir "${third_party_root}/sdl"
    CACHE PATH "Directory containing the SDL library.")
set(dependencies_libogg_cmake_dir "${CMAKE_MODULE_PATH}/libogg"
    CACHE PATH "Directory containing the libogg cmake project.")
set(dependencies_libogg_distr_dir "${third_party_root}/libogg"
    CACHE PATH "Directory containing the libogg library.")
set(dependencies_libvorbis_cmake_dir "${CMAKE_MODULE_PATH}/libvorbis"
    CACHE PATH "Directory containing the libvorbis cmake project.")
set(dependencies_libvorbis_distr_dir "${third_party_root}/libvorbis"
    CACHE PATH "Directory containing the libvorbis library.")
set(dependencies_sdl_mixer_cmake_dir "${CMAKE_MODULE_PATH}/sdl_mixer"
    CACHE PATH "Directory containing the SDL Mixer cmake project.")
set(dependencies_sdl_mixer_distr_dir "${third_party_root}/sdl_mixer"
    CACHE PATH "Directory containing the SDL Mixer library.")

# Option to enable / disable the test build.
option(pindrop_build_tests "Build tests for this project." ON)

# Option to only build flatc
option(pindrop_only_flatc "Only build FlatBuffers compiler." OFF)

# Include SDL in this project while working around SDL requirement of out of
# tree builds.
set(SDL_SHARED OFF CACHE BOOL "Build a shared version of the library")
# (hack) We redefine install to stop SDL's CMakeLists.txt from referring to SDL2 lib
function(install)
endfunction(install)
# For some reason SDL defaults to these not being defined, and then defines their own,
# causing linker errors.
add_definitions(-DHAVE_LIBC)
add_definitions(-DHAVE_STDIO_H)
set(BIN_DIR ${CMAKE_CURRENT_BINARY_DIR})
set(CMAKE_BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/.)
if(APPLE)
  set(SDL_AUDIO_DRIVER_COREAUDIO 1)
endif(APPLE)
if(NOT pindrop_only_flatc AND NOT TARGET SDL2-static)
  add_subdirectory(${dependencies_sdl_dir} ${tmp_dir}/sdl)
endif()
set(CMAKE_BINARY_DIR ${BIN_DIR})

# Include libogg.
if(NOT pindrop_only_flatc AND NOT TARGET libogg)
  add_subdirectory("${dependencies_libogg_cmake_dir}" ${tmp_dir}/libogg)
endif()

# Include libvorbis.
if(NOT pindrop_only_flatc AND NOT TARGET libvorbis)
  add_subdirectory("${dependencies_libvorbis_cmake_dir}" ${tmp_dir}/libvorbis)
endif()

# Include SDL Mixer.
set(SDL_MIXER_OGG_SUPPORT ON CACHE BOOL "Compile in Ogg Vorbis support")
if(NOT pindrop_only_flatc AND NOT TARGET sdl_mixer)
  add_subdirectory("${dependencies_sdl_mixer_cmake_dir}" ${tmp_dir}/sdl_mixer)
endif()

# Include MathFu in this project with test and benchmark builds disabled.
set(mathfu_build_benchmarks OFF CACHE BOOL "")
set(mathfu_build_tests OFF CACHE BOOL "")
add_subdirectory(${dependencies_mathfu_dir} ${tmp_dir}/mathfu)

# Include FlatBuffers in this project.
set(FLATBUFFERS_BUILD_TESTS OFF CACHE BOOL "")
if(NOT TARGET flatc)
  add_subdirectory("${dependencies_flatbuffers_dir}" ${tmp_dir}/flatbuffers)
endif()

# Generate source files for all FlatBuffers schema files under the src
# directory.
set(PINDROP_FLATBUFFERS_GENERATED_INCLUDES_DIR
    ${CMAKE_CURRENT_SOURCE_DIR}${CMAKE_FILES_DIRECTORY}/include)
set(PINDROP_FLATBUFFERS_GENERATED_INCLUDES_DIR
    ${PINDROP_FLATBUFFERS_GENERATED_INCLUDES_DIR} PARENT_SCOPE)
file(GLOB_RECURSE FLATBUFFERS_SCHEMAS ${CMAKE_CURRENT_SOURCE_DIR}/schemas/*.fbs)

# Generate rules to build the set of output files from the set of input
# schema files.
foreach(flatbuffers_schema ${FLATBUFFERS_SCHEMAS})
  get_filename_component(filename ${flatbuffers_schema} NAME_WE)
  set(flatbuffers_include
    ${PINDROP_FLATBUFFERS_GENERATED_INCLUDES_DIR}/${filename}_generated.h)
  add_custom_command(
    OUTPUT ${flatbuffers_include}
    COMMAND flatc --gen-includes
        -o ${PINDROP_FLATBUFFERS_GENERATED_INCLUDES_DIR}
        -c ${flatbuffers_schema}
    DEPENDS flatc ${flatbuffers_schema})
  list(APPEND FLATBUFFERS_GENERATED_INCLUDES ${flatbuffers_include})
endforeach()
add_custom_target(pindrop_generated_includes DEPENDS ${FLATBUFFERS_GENERATED_INCLUDES})

get_property(flatc_location TARGET flatc PROPERTY LOCATION)
add_custom_target(pindrop_demo_assets
  COMMAND python ${CMAKE_CURRENT_SOURCE_DIR}/scripts/build_assets.py
  --flatc ${flatc_location} DEPENDS flatc)

# AudioEngine source files.
set(pindrop_SRCS
    include/pindrop/pindrop.h
    src/audio_engine.cpp
    src/audio_engine_internal_state.h
    src/backend.cpp
    src/backend.h
    src/bus.cpp
    src/bus.h
    src/channel.cpp
    src/channel_internal_state.cpp
    src/channel_internal_state.h
    src/intrusive_list.h
    src/listener.cpp
    src/listener_internal_state.h
    src/main.cpp
    src/ref_counter.cpp
    src/ref_counter.h
    src/sound.cpp
    src/sound.h
    src/sound_bank.cpp
    src/sound_bank.h
    src/sound_collection.cpp
    src/sound_collection.h)

set(pindrop_demo_SRCS src/main.cpp)

# Includes for this project.
include_directories(src include)
if(WIN32)
  include_directories(external/include/windows)
endif()
include_directories(${PINDROP_FLATBUFFERS_GENERATED_INCLUDES_DIR})
include_directories(${dependencies_flatbuffers_dir}/include)
include_directories(${dependencies_webp_distr_dir}/include)
include_directories(${dependencies_webp_distr_dir}/src)

# SDL includes.
include_directories(${SDL2_BINARY_DIR}/include ${SDL2_SOURCE_DIR}/include)

# libogg includes.
include_directories(${dependencies_libogg_distr_dir}/include)

# libvorbis includes.
include_directories(${dependencies_libvorbis_distr_dir}/include)

# SDL Mixer includes.
include_directories(${dependencies_sdl_mixer_distr_dir})

# Detect clang
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  set(CMAKE_COMPILER_IS_CLANGXX 1)
endif()

# Compiler flags.
set(C_FLAGS_WARNINGS "")
if(MSVC)
  set(C_FLAGS_WARNINGS "/W4 /WX")
elseif(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX OR
       CMAKE_COMPILER_IS_CLANGXX)
  add_definitions(-g)
  set(CMAKE_CXX_FLAGS
    "${CMAKE_CXX_FLAGS} -std=c++0x -Wall -pedantic -Werror -Wextra -Wno-unused-function")
  # -Wno-unused-function is because of webp encode.h
endif()
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${C_FLAGS_WARNINGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${C_FLAGS_WARNINGS}")

set (SDL_LIBRARIES SDL2-static)
if(WIN32)
  add_definitions(-D_USE_MATH_DEFINES)
  set(SDL_LIBRARIES SDL2main ${SDL_LIBRARIES})
  link_directories("$ENV{DXSDK_DIR}/Lib/$ENV{PROCESSOR_ARCHITECTURE}")
endif()

# Executable target.
add_library(pindrop ${pindrop_SRCS})
mathfu_configure_flags(pindrop)

# Dependencies for the executable target.
add_dependencies(pindrop pindrop_generated_includes pindrop_demo_assets)

add_executable(pindrop_demo ${pindrop_demo_SRCS})
target_link_libraries(pindrop_demo
  pindrop
  ${SDL_LIBRARIES}
  sdl_mixer
  libvorbis
  libogg)
mathfu_configure_flags(pindrop_demo)

# gtest seems to prefer the non-DLL runtime on Windows, which conflicts with
# everything else.
option(
  gtest_force_shared_crt
  "Use shared (DLL) run-time lib even when Google Test is built as static lib."
  ON)

# Tests.
if(NOT pindrop_only_flatc AND pindrop_build_tests)
  set(GUNIT_INCDIR "${dependencies_gtest_dir}/include")
  set(GTEST_LIBDIR "${dependencies_gtest_dir}")
  add_subdirectory(${dependencies_gtest_dir} googletest)
  include(${GTEST_LIBDIR}/cmake/internal_utils.cmake)
  config_compiler_and_linker()
  string(REPLACE "-W4" "-W3" cxx_default "${cxx_default}")
  string(REPLACE "-Wshadow" "" cxx_default "${cxx_default}")
  string(REPLACE "-Wextra" "" cxx_default "${cxx_default}")

  include_directories(${GUNIT_INCDIR}
                      ${CMAKE_CURRENT_SOURCE_DIR}
                      ${dependencies_flatbuffers_dir}/include)

  # Common libraries for tests.
  if(NOT MSVC)
    find_package(Threads)
  endif()

  set (SDL_LIBRARIES SDL2-static)
  if(WIN32)
    set(SDL_LIBRARIES SDL2main;${SDL_LIBRARIES})
  endif()

  function(test_executable name libs)
    cxx_executable_with_flags(${name}_test "${cxx_default}"
        "${libs}" ${CMAKE_CURRENT_SOURCE_DIR}/src/${name}_test.cpp
        ${ARGN})
    mathfu_configure_flags(${name}_test)
  endfunction()

  test_executable(audio_engine "gtest;pindrop;${SDL_LIBRARIES}")
  test_executable(intrusive_list "gtest")
endif()

