# Copyright (c) 2017-present, Facebook, Inc.
# All rights reserved.
#
# This source code is licensed under the BSD-style license found in the
# LICENSE file in the root directory of this source tree. An additional grant
# of patent rights can be found in the PATENTS file in the same directory.

cmake_minimum_required (VERSION 2.8)
cmake_policy(VERSION 2.8)

# OpenMP support?
SET(WITH_OPENMP ON CACHE BOOL "OpenMP support if available?")
IF (APPLE AND CMAKE_COMPILER_IS_GNUCC)
  EXEC_PROGRAM (uname ARGS -v  OUTPUT_VARIABLE DARWIN_VERSION)
  STRING (REGEX MATCH "[0-9]+" DARWIN_VERSION ${DARWIN_VERSION})
  MESSAGE (STATUS "MAC OS Darwin Version: ${DARWIN_VERSION}")
  IF (DARWIN_VERSION GREATER 9)
    SET(APPLE_OPENMP_SUCKS 1)
  ENDIF (DARWIN_VERSION GREATER 9)
  EXECUTE_PROCESS (COMMAND ${CMAKE_C_COMPILER} -dumpversion
    OUTPUT_VARIABLE GCC_VERSION)
  IF (APPLE_OPENMP_SUCKS AND GCC_VERSION VERSION_LESS 4.6.2)
        MESSAGE(STATUS "Warning: Disabling OpenMP (unstable with this version of GCC)")
    MESSAGE(STATUS " Install GCC >= 4.6.2 or change your OS to enable OpenMP")
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wno-unknown-pragmas")
    SET(WITH_OPENMP OFF CACHE BOOL "OpenMP support if available?" FORCE)
  ENDIF ()
ENDIF ()

IF (WITH_OPENMP)
  FIND_PACKAGE(OpenMP)
  IF(OPENMP_FOUND)
    MESSAGE(STATUS "Compiling with OpenMP support")
    SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
  ENDIF(OPENMP_FOUND)
ENDIF (WITH_OPENMP)

FIND_PACKAGE(Torch REQUIRED)
IF(Torch_FOUND)
  ADD_DEFINITIONS(-DHAS_TORCH)
  ADD_DEFINITIONS(-std=c99)
ENDIF()

set(PKGNAME wav2letter)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/utils)
add_library(${PKGNAME} MODULE
  "utils/utils.c"
  "utils/viterbi.c"
  "utils/autosegcriterion.c"
)

IF(Torch_FOUND)
  TARGET_LINK_LIBRARIES(${PKGNAME} TH)
ENDIF()

install(TARGETS ${PKGNAME} LIBRARY
  DESTINATION ${LUA_CPATH})

FILE(GLOB luafiles *.lua)
install(FILES ${luafiles}
  DESTINATION ${LUA_PATH}/${PKGNAME})

install(FILES
  "utils/init.lua"
  DESTINATION ${LUA_PATH}/${PKGNAME}/utils)

set(LIBPKGNAME waveval)
set(LIBTESTPKGNAME wavevaltest)

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/lib)
add_library(${LIBPKGNAME} SHARED "libwaveval/waveval.c")
add_executable(${LIBTESTPKGNAME} ${CMAKE_CURRENT_SOURCE_DIR}/libwaveval/test/wavevaltest.c)

TARGET_LINK_LIBRARIES(${LIBPKGNAME} TH luaT ${LUA_LIB})
TARGET_LINK_LIBRARIES(${LIBTESTPKGNAME} ${LIBPKGNAME} ${LUA_LIB})

install(TARGETS ${LIBPKGNAME} LIBRARY
  DESTINATION ${LIB_DIR})
FILE(GLOB wavevalluafiles runtime/*.lua)
install(FILES ${wavevalluafiles}
  DESTINATION ${LUA_PATH}/${PKGNAME}/runtime)
