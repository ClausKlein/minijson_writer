# You don't need to compile any library to use minijson_writer in your project:
# just include the minijson_writer.hpp header anywhere you need, and you're ready to go.

# Use this cmake project to build and run the unit tests (you will need Google Test):
#   $ mkdir build
#   $ cd build
#   $ cmake ..
#   $ make
#   $ ctest
#   $ make install

cmake_minimum_required(VERSION 3.13...3.21)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

project(minijson_writer LANGUAGES CXX)

option(CMAKE_CXX_EXTENSIONS "Default value for CXX_EXTENSIONS property of targets." NO)
set(CMAKE_CXX_STANDARD 17 CACHE STRING "The used c++ standard")
set(CMAKE_CXX_STANDARD_REQUIRED On CACHE STRING "The c++ standard is requrired")

# Where to put all the RUNTIME targets when built.  This variable is used to
# initialize the RUNTIME_OUTPUT_DIRECTORY property on all the targets.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)

# ---- dependency ----
#XXX find_package(GTest 1.11 REQUIRED)
include(cmake/CPM.cmake)

CPMAddPackage(
  NAME googletest
  GITHUB_REPOSITORY google/googletest
  GIT_TAG release-1.11.0
  VERSION 1.11.0
  OPTIONS "BUILD_GMOCK OFF" "gtest_force_shared_crt"
)

# ---- Create library ----

# Note: for header-only libraries change all PUBLIC flags to INTERFACE
add_library(${PROJECT_NAME} INTERFACE minijson_writer.hpp)
set_target_properties(${PROJECT_NAME} PROPERTIES CXX_STANDARD ${CMAKE_CXX_STANDARD})
target_include_directories(
  ${PROJECT_NAME} BEFORE INTERFACE $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}>
                                   $<INSTALL_INTERFACE:include>
)
# being a cross-platform target, we enforce standards conformance on MSVC
target_compile_options(${PROJECT_NAME} INTERFACE "$<$<COMPILE_LANG_AND_ID:CXX,MSVC>:/permissive->")

add_library(minijson::${PROJECT_NAME} ALIAS ${PROJECT_NAME})

# ---- Create executables ----

add_executable(minijson_writer_tests minijson_writer_tests.cpp minijson_writer.hpp)
target_link_libraries(minijson_writer_tests minijson::${PROJECT_NAME} GTest::gtest_main)

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang" AND CMAKE_CXX_SIMULATE_ID MATCHES "MSVC")
  target_compile_options(${PROJECT_NAME}_tests PRIVATE /W4 /WX)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  target_compile_options(${PROJECT_NAME}_tests PRIVATE -Wall -Wextra -Werror -Wno-shadow -Wno-unknown-pragmas)

  if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND CMAKE_CXX_COMPILER_VERSION GREATER_EQUAL 10.0)
    target_compile_options(${PROJECT_NAME}_tests PRIVATE -Wdeprecated-copy-dtor -Wnewline-eof)
  endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
  string(REGEX REPLACE " /W[0-4]" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
  target_compile_options(${PROJECT_NAME}_tests PRIVATE /W4 /WX)
endif()

# ---- code coverage ----

if(NOT MSVC)
  option(ENABLE_TEST_COVERAGE "Enable test coverage" ON)
  if(ENABLE_TEST_COVERAGE AND "${CMAKE_BUILD_TYPE}" STREQUAL Debug)
    target_compile_options(minijson_writer_tests PUBLIC -O0 -g -fprofile-arcs -ftest-coverage)
    target_link_options(minijson_writer_tests PUBLIC -fprofile-arcs -ftest-coverage)
  endif()
endif()

# ---- ctest ----

enable_testing()
add_test(NAME ${PROJECT_NAME} COMMAND minijson_writer_tests)

# ---- install ----

install(FILES minijson_writer.hpp DESTINATION include)

