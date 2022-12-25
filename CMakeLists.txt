#
# Copyright (c) 2022 INRIA
#

cmake_minimum_required(VERSION 3.14)

set(PROJECT_NAME proxsuite)

option(INITIALIZE_EIGEN_WITH_NAN "Initialize Eigen objects with NAN values" OFF)
option(CHECK_RUNTIME_MALLOC
       "Check if some memory allocations are performed at runtime" OFF)
option(SUFFIX_SO_VERSION "Suffix library name with its version" ON)

option(BUILD_WITH_VECTORIZATION_SUPPORT
       "Build the library with the support of modern SIMD instructions." ON)
option(BUILD_BINDINGS_WITH_AVX2_SUPPORT "Build the bindings with AVX2 support."
       ON)
option(BUILD_BINDINGS_WITH_AVX512_SUPPORT
       "Build the bindings with AVX512 support." ON)
option(TEST_JULIA_INTERFACE "Run the julia examples as unittest" OFF)

message(STATUS "CMAKE_SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR}")
if(INITIALIZE_EIGEN_WITH_NAN)
  add_definitions(-DEIGEN_INITIALIZE_MATRICES_BY_NAN)
endif(INITIALIZE_EIGEN_WITH_NAN)
if(CHECK_RUNTIME_MALLOC)
  message(STATUS "Check if some memory allocations are performed at runtime.")
  add_definitions(-DPROXSUITE_EIGEN_CHECK_MALLOC)
  add_definitions(-DEIGEN_RUNTIME_NO_MALLOC)
endif(CHECK_RUNTIME_MALLOC)

# set CXX standard
if(DEFINED CMAKE_CXX_STANDARD)
  check_minimal_cxx_standard(14 ENFORCE)
else()
  set(CMAKE_CXX_STANDARD 17)
endif()

# Handle Windows context
if(MSVC)
  # add_definitions(-D_USE_MATH_DEFINES)
  add_definitions(-DNOMINMAX)
endif()

# Look for dependencies
find_package(Eigen3 REQUIRED)

set(SIMDE_HINT_FAILURE
    "Set BUILD_WITH_VECTORIZATION_SUPPORT=OFF or install Simde on your system.\n If Simde is already installed, ensure that the CMake variable CMAKE_MODULE_PATH correctly points toward the location of FindSimde.cmake file."
)
if(BUILD_WITH_VECTORIZATION_SUPPORT)
  include(FetchContent)
  # set(FETCHCONTENT_UPDATES_DISCONNECTED ON)

  FetchContent_Declare(
      simde
      GIT_REPOSITORY https://github.com/simd-everywhere/simde.git
      GIT_TAG v0.7.2
      GIT_SHALLOW    TRUE
  )
  FetchContent_MakeAvailable(simde)
  set(Simde_INCLUDE_DIR ${simde_SOURCE_DIR})
endif()

# Build the main library
file(GLOB_RECURSE ${PROJECT_NAME}_HEADERS ${PROJECT_SOURCE_DIR}/include/*.hpp)

add_library(proxsuite INTERFACE)
if(MSVC)
  target_compile_options(proxsuite INTERFACE /permissive-)
  target_compile_options(proxsuite INTERFACE $<$<COMPILE_LANGUAGE:CXX>:/bigobj>)
endif(MSVC)
target_link_libraries(
  proxsuite
  PUBLIC
  INTERFACE Eigen3::Eigen)
target_include_directories(
  proxsuite INTERFACE "$<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>"
                      "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")
target_include_directories(
  proxsuite INTERFACE "$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>")
set(EXPORTED_TARGETS_LIST proxsuite)

add_header_group(${PROJECT_NAME}_HEADERS)

if(BUILD_WITH_VECTORIZATION_SUPPORT)
  add_library(proxsuite-vectorized INTERFACE)
  target_link_libraries(
    proxsuite-vectorized
    PUBLIC
    INTERFACE proxsuite)
  target_link_libraries(
    proxsuite-vectorized
    PUBLIC
    INTERFACE ${Simde_INCLUDE_DIR})
  target_compile_definitions(proxsuite-vectorized INTERFACE PROXSUITE_VECTORIZE)
  list(APPEND EXPORTED_TARGETS_LIST proxsuite-vectorized)
endif()

