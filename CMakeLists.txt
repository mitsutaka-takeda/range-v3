cmake_minimum_required(VERSION 3.3)
set(CMAKE_LEGACY_CYGWIN_WIN32 0)

project(Range-v3)

find_package(Doxygen)
find_package(Git)

# Select C++ standard to be used for compiling the tests,
# for example: 11, 14, 17, 1z, 1y, ...
#
if(RANGES_CXX_STD)
else()
  # Defaults to C++11 if not set:
  set(RANGES_CXX_STD 11)
endif()

enable_testing()
include(CTest)
find_program(MEMORYCHECK_COMMAND valgrind)
if(MEMORYCHECK_COMMAND-NOTFOUND)
  message("[W] Valgrind not found")
else()
  message("Valgrind: ${MEMORYCHECK_COMMAND}")
  set(MEMORYCHECK_COMMAND_OPTIONS "--trace-children=yes --leak-check=full")
endif()

add_library(
  range-v3
  INTERFACE
  )
target_include_directories(
  range-v3
  INTERFACE
  ${CMAKE_CURRENT_SOURCE_DIR}/include
  )

# Clang/C2 will blow up with various parts of the standard library
# if compiling with -std less than c++14.
if(("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang") AND
   ("x${CMAKE_CXX_SIMULATE_ID}x" STREQUAL "xMSVCx") AND
   ("${RANGES_CXX_STD}" STREQUAL "11"))
  set(CMAKE_CXX_STANDARD 14)
  set(RANGES_CXX_STD 14)
endif()

if("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
  add_compile_options(
    -std=c++${RANGES_CXX_STD}
    -ftemplate-backtrace-limit=0
    -Weverything
    -Werror
    -pedantic-errors
    -Wno-error=deprecated-declarations
    -Wno-error=deprecated
    -Wno-c++98-compat
    -Wno-c++98-compat-pedantic
    -Wno-return-stack-address
    -Wno-unused-variable
    -Wno-unused-parameter
    -Wno-weak-vtables
    -Wno-padded
    -Wno-global-constructors
    -Wno-exit-time-destructors
    -Wno-shorten-64-to-32
    -Wno-sign-conversion
    -Wno-missing-prototypes
    -Wno-missing-variable-declarations
    -Wno-shadow
    -Wno-old-style-cast
    -Wno-documentation
    -Wno-documentation-unknown-command
    $<$<CONFIG:Debug>:-O0>
    $<$<CONFIG:Debug>:-fno-inline>
    $<$<CONFIG:Debug>:-g3>
    $<$<CONFIG:Debug>:-fstack-protector-all>
    $<$<CONFIG:Release>:-Ofast>
    $<$<CONFIG:Release>:-g0>
    $<$<CONFIG:Release>:-march=native>
    $<$<CONFIG:Release>:-mtune=native>
    $<$<CONFIG:Release>:-DNDEBUG>
    )
elseif(CMAKE_COMPILER_IS_GNUCXX)
  add_compile_options(
    -std=c++${RANGES_CXX_STD}
    -ftemplate-backtrace-limit=0
    -Wall
    -Wextra
    -pedantic
    -Werror
    -pedantic-errors
    -Wno-error=deprecated-declarations
    -Wno-missing-field-initializers
    -Wno-unused-parameter
    $<$<CONFIG:Debug>:-O0>
    $<$<CONFIG:Debug>:-fno-inline>
    $<$<CONFIG:Debug>:-g3>
    $<$<CONFIG:Debug>:-fstack-protector-all>
    $<$<CONFIG:Release>:-Ofast>
    $<$<CONFIG:Release>:-g0>
    $<$<CONFIG:Release>:-march=native>
    $<$<CONFIG:Release>:-mtune=native>
    $<$<CONFIG:Release>:-DNDEBUG>
    )
# else()
#   message(FATAL_ERROR "Unknown compiler.")
endif()

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

add_subdirectory(doc)

if(BIICODE)
    ADD_BIICODE_TARGETS()

    string(REPLACE " " ";" TARGET_CXX_FLAGS         ${CMAKE_CXX_FLAGS})
    string(REPLACE " " ";" TARGET_CXX_FLAGS_DEBUG   ${CMAKE_CXX_FLAGS_DEBUG})
    string(REPLACE " " ";" TARGET_CXX_FLAGS_RELEASE ${CMAKE_CXX_FLAGS_RELEASE})

    target_compile_options(${BII_BLOCK_TARGET} INTERFACE ${TARGET_CXX_FLAGS}
                           $<$<CONFIG:Debug>:${TARGET_CXX_FLAGS_DEBUG}>
                           $<$<CONFIG:Release>:${TARGET_CXX_FLAGS_RELEASE}>)

    return()
endif()

#   range_v3_list_remove_glob(<list> <GLOB|GLOB_RECURSE> [globbing expressions]...)
#
# Generates a list of files matching the given glob expressions, and remove
# the matched elements from the given <list>.
#
# Adapted from Boost.Hana: https://github.com/ldionne/hana/
#
macro(range_v3_list_remove_glob list glob)
    file(${glob} _bhlrg10321023_avoid_macro_clash_matches ${ARGN})
    list(REMOVE_ITEM ${list} ${_bhlrg10321023_avoid_macro_clash_matches})
endmacro()

# Test all headers
if(NOT RANGE_V3_NO_HEADER_CHECK)
  include(CheckIncludeFileCXX)
  file(GLOB_RECURSE RANGE_V3_PUBLIC_HEADERS "${CMAKE_CURRENT_SOURCE_DIR}/include/*.hpp")
  range_v3_list_remove_glob(RANGE_V3_PUBLIC_HEADERS GLOB_RECURSE
    "${CMAKE_CURRENT_SOURCE_DIR}/include/range/v3/detail/re_enable_warnings.hpp"
    )

  set(CMAKE_REQUIRED_INCLUDES "${CMAKE_CURRENT_SOURCE_DIR}/include")
  set(CMAKE_REQUIRED_FLAGS "-std=c++${RANGES_CXX_STD}")
  foreach(_header IN LISTS RANGE_V3_PUBLIC_HEADERS)
    file(RELATIVE_PATH _relative "${CMAKE_CURRENT_SOURCE_DIR}/include" "${_header}")
    check_include_file_cxx("${_relative}" ${_relative}_FOUND)
    if(NOT ${_relative}_FOUND)
      message(FATAL_ERROR "Compilation of header ${_relative} failed")
    endif()
  endforeach()
endif()

add_subdirectory(test)
add_subdirectory(example)
add_subdirectory(perf)
add_subdirectory(scratch)

install(DIRECTORY include/ DESTINATION include
        FILES_MATCHING PATTERN "*.hpp")
