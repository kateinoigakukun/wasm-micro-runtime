# Copyright (C) 2019 Intel Corporation. All rights reserved.
# SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception

cmake_minimum_required (VERSION 3.0)

project(bwa_wasm)

################  EMCC ################
if(NOT DEFINED ENV{EMSDK})
  message(FATAL_ERROR
    "can not find emsdk. "
    "please refer to https://emscripten.org/docs/getting_started/downloads.html "
    "and install it, "
    "or active emsdk by 'source ./emsdk_env.sh'"
  )
endif()

################  BINARYEN ################
find_program(WASM_OPT
    NAMES wasm-opt
    PATHS /opt/binaryen-version_97/bin /opt/binaryen/bin
)

if (NOT WASM_OPT)
  message(FATAL_ERROR
    "can not find wasm-opt. "
    "please download it from "
    "https://github.com/WebAssembly/binaryen/releases/download/version_97/binaryen-version_97-x86_64-linux.tar.gz "
    "and install it under /opt"
  )
endif()

#######################################
include(ExternalProject)

################ HEADERS ################
ExternalProject_Add(headers_from_emcc
   PREFIX headers
   SOURCE_DIR "$ENV{EMSDK}/upstream/emscripten/system/include/SSE"
   CONFIGURE_COMMAND ""
   BUILD_COMMAND  ""
   INSTALL_COMMAND mkdir -p ${CMAKE_CURRENT_SOURCE_DIR}/include/SSE
                     && ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/include/pthread/sys
                     && ${CMAKE_COMMAND} -E make_directory ${CMAKE_CURRENT_SOURCE_DIR}/include/emscripten
                     # copy emscripten SSE header files
                     && ${CMAKE_COMMAND} -E copy $ENV{EMSDK}/upstream/emscripten/system/include/SSE/immintrin.h ${CMAKE_CURRENT_SOURCE_DIR}/include/SSE/
                     # SSE
                     && ${CMAKE_COMMAND} -E copy $ENV{EMSDK}/upstream/emscripten/system/include/SSE/xmmintrin.h ${CMAKE_CURRENT_SOURCE_DIR}/include/SSE/
                     # SSE2
                     && ${CMAKE_COMMAND} -E copy $ENV{EMSDK}/upstream/emscripten/system/include/SSE/emmintrin.h ${CMAKE_CURRENT_SOURCE_DIR}/include/SSE/
                     # SSE4.1
                     && ${CMAKE_COMMAND} -E copy $ENV{EMSDK}/upstream/emscripten/system/include/SSE/smmintrin.h ${CMAKE_CURRENT_SOURCE_DIR}/include/SSE/
                     # a fake empty header to aovid further depenency
                     && ${CMAKE_COMMAND} -E touch ${CMAKE_CURRENT_SOURCE_DIR}/include/emscripten/emscripten.h
                     # copy emscripten pthread related header files
                     && ${CMAKE_COMMAND} -E copy $ENV{EMSDK}/upstream/emscripten/system/include/libc/pthread.h ${CMAKE_CURRENT_SOURCE_DIR}/include/pthread/
                     && ${CMAKE_COMMAND} -E copy $ENV{EMSDK}/upstream/emscripten/system/include/libc/signal.h ${CMAKE_CURRENT_SOURCE_DIR}/include/pthread/
                     && ${CMAKE_COMMAND} -E copy $ENV{EMSDK}/upstream/emscripten/system/include/libc/netdb.h ${CMAKE_CURRENT_SOURCE_DIR}/include/pthread/
                     && ${CMAKE_COMMAND} -E copy $ENV{EMSDK}/upstream/emscripten/system/include/libc/sys/wait.h ${CMAKE_CURRENT_SOURCE_DIR}/include/pthread/sys/
                     && ${CMAKE_COMMAND} -E copy $ENV{EMSDK}/upstream/emscripten/system/include/libc/sys/socket.h ${CMAKE_CURRENT_SOURCE_DIR}/include/pthread/sys/
)

################ libz ################
ExternalProject_Add(libz_src
  PREFIX libz
  GIT_REPOSITORY https://github.com/madler/zlib.git
  GIT_TAG        master
  GIT_PROGRESS   ON
  GIT_SHALLOW    ON
  SOURCE_DIR     ${CMAKE_CURRENT_SOURCE_DIR}/libz
  CONFIGURE_COMMAND ""
  BUILD_COMMAND  ""
  INSTALL_COMMAND ""
)

################ bwa ################
ExternalProject_Add(bwa
  PREFIX bwa
  GIT_REPOSITORY https://github.com/lh3/bwa.git
  GIT_TAG        master
  GIT_PROGRESS   ON
  GIT_SHALLOW    ON
  SOURCE_DIR     ${CMAKE_CURRENT_SOURCE_DIR}/bwa
  DEPENDS        libz_src headers_from_emcc
  UPDATE_COMMAND git clean -fd && git checkout -- *
                  && ${CMAKE_COMMAND} -E echo "Copying pre-installed CMakeLists.txt"
                  && ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/CMakeLists.bwa_wasm.txt CMakeLists.txt
  CONFIGURE_COMMAND  ${CMAKE_COMMAND} -DCMAKE_TOOLCHAIN_FILE=${CMAKE_CURRENT_SOURCE_DIR}/../cmake/toolchain.cmake ${CMAKE_CURRENT_SOURCE_DIR}/bwa
  BUILD_COMMAND  make bwa_wasm_opt
  INSTALL_COMMAND ${CMAKE_COMMAND} -E copy ./bwa.opt.wasm ${CMAKE_CURRENT_SOURCE_DIR}/build/bwa.wasm
)
