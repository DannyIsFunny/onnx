# Minimum CMake required
cmake_minimum_required(VERSION 3.1)
include(cmake/Utils.cmake)
# Set default build type
if(NOT CMAKE_BUILD_TYPE)
  message(STATUS "Build type not set - defaulting to Release")
  set(
    CMAKE_BUILD_TYPE "Release"
    CACHE
      STRING
      "Choose the type of build from: Debug Release RelWithDebInfo MinSizeRel Coverage."
    FORCE)
endif()
cmake_policy(SET CMP0063 NEW)

# Project
project(onnx C CXX)
option(ONNX_BUILD_BENCHMARKS "Build ONNX micro-benchmarks" OFF)
option(ONNX_USE_PROTOBUF_SHARED_LIBS "Build ONNX using protobuf shared library. Sets PROTOBUF_USE_DLLS CMAKE Flag and Protobuf_USE_STATIC_LIBS. " OFF)

option(BUILD_ONNX_PYTHON "Build Python binaries" OFF)
option(ONNX_GEN_PB_TYPE_STUBS "Generate protobuf python type stubs" ON)
option(ONNX_WERROR "Build with Werror" OFF)
option(ONNX_COVERAGE "Build with coverage instrumentation" OFF)
option(ONNX_BUILD_TESTS "Build ONNX C++ APIs Tests" OFF)
option(ONNX_USE_LITE_PROTO "Use lite protobuf instead of full." OFF)
option(ONNX_DISABLE_EXCEPTIONS "Disable exception handling." OFF)
option(ONNX_DISABLE_STATIC_REGISTRATION "Disable static registration for onnx operator schemas." OFF)

if(NOT DEFINED ONNX_ML)
  if(DEFINED ENV{ONNX_ML})
    set(DEFAULT_ONNX_ML $ENV{ONNX_ML})
  else()
    set(DEFAULT_ONNX_ML ON)
  endif()
  option(ONNX_ML "Enable traditional ML API." ${DEFAULT_ONNX_ML})
endif()

if(NOT DEFINED ONNX_VERIFY_PROTO3)
  if(DEFINED ENV{ONNX_VERIFY_PROTO3})
    set(PROTO3_ENABLED $ENV{ONNX_VERIFY_PROTO3})
  else()
    set(PROTO3_ENABLED OFF)
  endif()
  option(ONNX_VERIFY_PROTO3 "Generate code by proto3" ${PROTO3_ENABLED})
endif()

if(ONNX_USE_PROTOBUF_SHARED_LIBS)
  if(MSVC)
    #TODO: if ONNX_USE_MSVC_STATIC_RUNTIME is ON, it may not work
    add_definitions(-DPROTOBUF_USE_DLLS)
  endif()
  set(Protobuf_USE_STATIC_LIBS OFF)
else()
  set(Protobuf_USE_STATIC_LIBS ON)
endif()

# Required to use /std:c++17 or higher on Windows
# For other platforms, set C++11 as standard for the whole project
if(NOT MSVC)
  set(CMAKE_CXX_STANDARD 11)
else()
  string(APPEND CMAKE_CXX_FLAGS " /std:c++17")
endif()

include(GNUInstallDirs)

set(ONNX_ROOT ${PROJECT_SOURCE_DIR})

# Read ONNX version
file(READ "${PROJECT_SOURCE_DIR}/VERSION_NUMBER" ONNX_VERSION)
string(STRIP "${ONNX_VERSION}" ONNX_VERSION)

if(NOT MSVC)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wnon-virtual-dtor")
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -O0")
  if(ONNX_COVERAGE)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fprofile-arcs -ftest-coverage")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs -ftest-coverage")
  endif()
endif()

if(NOT ONNX_NAMESPACE)
  set(ONNX_NAMESPACE "onnx")
endif()

if(MSVC)
  if(NOT ONNX_DISABLE_EXCEPTIONS)
    string(APPEND CMAKE_CXX_FLAGS " /EHsc /wd26812")
    string(APPEND CMAKE_C_FLAGS " /EHsc /wd26812")
  endif()
endif()

if(ONNX_DISABLE_EXCEPTIONS)
  add_compile_definitions("ONNX_NO_EXCEPTIONS")
  # Disable C++ exceptions.
  if(MSVC)
    string(REGEX REPLACE "/EHsc" "/EHs-c-" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    add_definitions(-D_HAS_EXCEPTIONS=0)
  else()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-exceptions -fno-unwind-tables -fno-asynchronous-unwind-tables")
  endif()
endif()

if(ONNX_WERROR)
  if(MSVC)
    string(APPEND CMAKE_CXX_FLAGS " /WX")
    string(APPEND CMAKE_C_FLAGS " /WX")
  else()
    string(APPEND CMAKE_CXX_FLAGS " -Werror")
    string(APPEND CMAKE_C_FLAGS " -Werror -Wno-error=incompatible-pointer-types")
  endif()
endif()

# find_package Python has replaced PythonInterp and PythonLibs since cmake 3.12
# Use the following command in the future; now this is only compatible with the latest pybind11
# find_package(Python ${PY_VERSION} COMPONENTS Interpreter Development REQUIRED)
find_package(PythonInterp ${PY_VERSION} REQUIRED)
find_package(PythonLibs ${PY_VERSION})

if(CMAKE_SYSTEM_NAME STREQUAL "AIX")
  set(CMAKE_NO_SYSTEM_FROM_IMPORTED 1)
endif()

if(ONNX_BUILD_TESTS)
  list(APPEND CMAKE_MODULE_PATH ${ONNX_ROOT}/cmake/external)
  include(googletest)
endif()

if((ONNX_USE_LITE_PROTO AND TARGET protobuf::libprotobuf-lite) OR ((NOT ONNX_USE_LITE_PROTO) AND TARGET protobuf::libprotobuf))
  # Sometimes we need to use protoc compiled for host architecture while linking
  # libprotobuf against target architecture. See https://github.com/caffe2/caffe
  # 2/blob/96f35ad75480b25c1a23d6e9e97bccae9f7a7f9c/cmake/ProtoBuf.cmake#L92-L99
  if(EXISTS "${ONNX_CUSTOM_PROTOC_EXECUTABLE}")
    message(STATUS "Using custom protoc executable")
    set(ONNX_PROTOC_EXECUTABLE ${ONNX_CUSTOM_PROTOC_EXECUTABLE})
  else()
    set(ONNX_PROTOC_EXECUTABLE $<TARGET_FILE:protobuf::protoc>)
  endif()
else()
  # Customized version of find Protobuf. We need to avoid situations mentioned
  # in https://github.com/caffe2/caffe2/blob/b7d983f255ef5496474f1ea188edb5e0ac4
  # 42761/cmake/ProtoBuf.cmake#L82-L92 The following section is stolen from
  # cmake/ProtoBuf.cmake in Caffe2
  find_program(Protobuf_PROTOC_EXECUTABLE
               NAMES protoc
               DOC "The Google Protocol Buffers Compiler")

  # Only if protoc was found, seed the include directories and libraries. We
  # assume that protoc is installed at PREFIX/bin. We use get_filename_component
  # to resolve PREFIX.
  if(Protobuf_PROTOC_EXECUTABLE)
    set(ONNX_PROTOC_EXECUTABLE ${Protobuf_PROTOC_EXECUTABLE})
    get_filename_component(_PROTOBUF_INSTALL_PREFIX
                           ${Protobuf_PROTOC_EXECUTABLE} DIRECTORY)
    get_filename_component(_PROTOBUF_INSTALL_PREFIX
                           ${_PROTOBUF_INSTALL_PREFIX}/.. REALPATH)
    find_library(Protobuf_PROTOC_LIBRARY
                 NAMES protoc
                 PATHS ${_PROTOBUF_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}
                 NO_DEFAULT_PATH)
    if(ONNX_USE_LITE_PROTO)
      find_library(Protobuf_LITE_LIBRARY
        NAMES protobuf-lite
        PATHS ${_PROTOBUF_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}
        NO_DEFAULT_PATH)
    else()
      find_library(Protobuf_LIBRARY
        NAMES protobuf
        PATHS ${_PROTOBUF_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}
        NO_DEFAULT_PATH)
    endif(ONNX_USE_LITE_PROTO)
    find_path(Protobuf_INCLUDE_DIR google/protobuf/service.h
              PATHS ${_PROTOBUF_INSTALL_PREFIX}/include
              NO_DEFAULT_PATH)
    find_package(Protobuf REQUIRED)
  endif()
endif()

# Build the libraries with -fPIC
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# function(RELATIVE_PROTOBUF_GENERATE_CPP SRCS HDRS ROOT_DIR) from https://githu
# b.com/tensorflow/tensorflow/blob/d2c3b873c6f8ff999a2e4ee707a84ff00d9c15a5/tens
# orflow/contrib/cmake/tf_core_framework.cmake to solve the problem that
# customized dir can't be specified when calling PROTOBUF_GENERATE_CPP.
function(RELATIVE_PROTOBUF_GENERATE_CPP NAME SRCS HDRS ROOT_DIR DEPEND)
  if(NOT ARGN)
    message(
      SEND_ERROR
        "Error: RELATIVE_PROTOBUF_GENERATE_CPP() called without any proto files"
      )
    return()
  endif()

  # Add ONNX_API prefix to protobuf classes and methods in all cases
  set(ONNX_DLLEXPORT_STR "dllexport_decl=ONNX_API:")

  set(${SRCS})
  set(${HDRS})

  set(GEN_PROTO_PY ${ROOT_DIR}/onnx/gen_proto.py)
  foreach(INFILE ${ARGN})
    set(ABS_FILE ${ROOT_DIR}/${INFILE})
    get_filename_component(FILE_DIR ${ABS_FILE} DIRECTORY)
    get_filename_component(FILE_WE ${INFILE} NAME_WE)
    # "onnx-data" check is because we do not want to create/compile an "onnx-data-ml.proto" file
    if(ONNX_ML AND NOT(FILE_WE STREQUAL "onnx-data"))
      if(ONNX_NAMESPACE STREQUAL "onnx")
        set(GENERATED_FILE_WE "${FILE_WE}-ml")
      else()
        set(GENERATED_FILE_WE "${FILE_WE}_${ONNX_NAMESPACE}-ml")
      endif()
    else()
      if(ONNX_NAMESPACE STREQUAL "onnx")
        set(GENERATED_FILE_WE "${FILE_WE}")
      else()
        set(GENERATED_FILE_WE "${FILE_WE}_${ONNX_NAMESPACE}")
      endif()
    endif()
    file(RELATIVE_PATH REL_DIR ${ROOT_DIR} ${FILE_DIR})
    set(OUTPUT_PROTO_DIR "${CMAKE_CURRENT_BINARY_DIR}/${REL_DIR}")

    set(OUTPUT_PB_HEADER "${OUTPUT_PROTO_DIR}/${GENERATED_FILE_WE}.pb.h")
    set(OUTPUT_PB_SRC "${OUTPUT_PROTO_DIR}/${GENERATED_FILE_WE}.pb.cc")
    set(GENERATED_PROTO "${OUTPUT_PROTO_DIR}/${GENERATED_FILE_WE}.proto")
    if(NOT (ONNX_NAMESPACE STREQUAL "onnx"))
      # We need this dummy header generated by gen_proto.py when ONNX_NAMESPACE
      # is not onnx
      list(APPEND ${HDRS} "${OUTPUT_PROTO_DIR}/${GENERATED_FILE_WE}.pb.h")
    endif()
    list(APPEND ${SRCS} "${OUTPUT_PB_SRC}")
    list(APPEND ${HDRS} "${OUTPUT_PB_HEADER}")

    if(NOT EXISTS "${OUTPUT_PROTO_DIR}")
      file(MAKE_DIRECTORY "${OUTPUT_PROTO_DIR}")
    endif()

    set(GEN_PROTO_ARGS
        -p
        "${ONNX_NAMESPACE}"
        -o
        "${OUTPUT_PROTO_DIR}"
        "${FILE_WE}")
    if(ONNX_ML)
      list(APPEND GEN_PROTO_ARGS -m)
    endif()
    if(ONNX_USE_LITE_PROTO)
      list(APPEND GEN_PROTO_ARGS -l)
    endif()
    if(ONNX_VERIFY_PROTO3)
        if(NOT ONNX_PROTOC_EXECUTABLE)
          message(FATAL_ERROR "Protobuf compiler not found")
        endif()
        list(APPEND GEN_PROTO_ARGS --protoc_path)
        list(APPEND GEN_PROTO_ARGS "${ONNX_PROTOC_EXECUTABLE}")
    endif()

    add_custom_command(OUTPUT "${GENERATED_PROTO}"
                       COMMAND "${PYTHON_EXECUTABLE}" "${GEN_PROTO_PY}"
                               ARGS ${GEN_PROTO_ARGS}
                       DEPENDS ${INFILE}
                       COMMENT "Running gen_proto.py on ${INFILE}"
                       VERBATIM)
    message("Generated: ${GENERATED_PROTO}")
    set(PROTOC_ARGS
        ${GENERATED_PROTO}
        -I
        ${CMAKE_CURRENT_BINARY_DIR}
        --cpp_out
        ${ONNX_DLLEXPORT_STR}${CMAKE_CURRENT_BINARY_DIR})
    if(BUILD_ONNX_PYTHON)
      list(APPEND PROTOC_ARGS --python_out
                  ${CMAKE_CURRENT_BINARY_DIR})
      if(ONNX_GEN_PB_TYPE_STUBS)
        # Haven't figured out how to generate mypy stubs on Windows yet
        if(NOT WIN32)
          # use PYTHON_EXECUTABLE to python protoc-gen-mypy.py
          configure_file(
            ${ROOT_DIR}/tools/protoc-gen-mypy.sh.in
            ${PROJECT_BINARY_DIR}/tools/protoc-gen-mypy.sh
            @ONLY)
          execute_process(COMMAND chmod +x ${PROJECT_BINARY_DIR}/tools/protoc-gen-mypy.sh)
          configure_file(
            ${ROOT_DIR}/tools/protoc-gen-mypy.py
            ${PROJECT_BINARY_DIR}/tools/protoc-gen-mypy.py
            COPYONLY)
          set(PROTOC_MYPY_PLUGIN_FILE ${PROJECT_BINARY_DIR}/tools/protoc-gen-mypy.sh)
        else()
          set(PROTOC_MYPY_PLUGIN_FILE ${ROOT_DIR}/tools/protoc-gen-mypy.bat)
        endif()
        list(APPEND PROTOC_ARGS
                    --plugin
                    protoc-gen-mypy=${PROTOC_MYPY_PLUGIN_FILE}
                    --mypy_out
                    ${ONNX_DLLEXPORT_STR}${CMAKE_CURRENT_BINARY_DIR})
      endif()
    endif()
    if(NOT ONNX_PROTOC_EXECUTABLE)
      message(FATAL_ERROR "Protobuf compiler not found")
    endif()
    if(ONNX_PROTO_POST_BUILD_SCRIPT)
      add_custom_command(
        OUTPUT "${OUTPUT_PB_SRC}" "${OUTPUT_PB_HEADER}"
        COMMAND ${ONNX_PROTOC_EXECUTABLE} ARGS ${PROTOC_ARGS}
        COMMAND "${CMAKE_COMMAND}" -DFILENAME=${OUTPUT_PB_HEADER}
                -DNAMESPACES=${ONNX_NAMESPACE} -P
                ${ONNX_PROTO_POST_BUILD_SCRIPT}
        COMMAND "${CMAKE_COMMAND}" -DFILENAME=${OUTPUT_PB_SRC}
                -DNAMESPACES=${ONNX_NAMESPACE} -P
                ${ONNX_PROTO_POST_BUILD_SCRIPT}
        DEPENDS ${GENERATED_PROTO} ${DEPEND}
        COMMENT "Running C++ protocol buffer compiler on ${GENERATED_PROTO}"
        VERBATIM)
    else()
      add_custom_command(
        OUTPUT "${OUTPUT_PB_SRC}" "${OUTPUT_PB_HEADER}"
        COMMAND ${ONNX_PROTOC_EXECUTABLE} ARGS ${PROTOC_ARGS}
        DEPENDS ${GENERATED_PROTO} ${DEPEND}
        COMMENT "Running C++ protocol buffer compiler on ${GENERATED_PROTO}"
        VERBATIM)
    endif()
    add_custom_target(${NAME} DEPENDS ${OUTPUT_PB_SRC} ${OUTPUT_PB_HEADER})
  endforeach()

  set_source_files_properties(${${SRCS}} ${${HDRS}} PROPERTIES GENERATED TRUE)
  set(${SRCS} ${${SRCS}} PARENT_SCOPE)
  set(${HDRS} ${${HDRS}} PARENT_SCOPE)
endfunction()

relative_protobuf_generate_cpp(gen_onnx_proto
                               __tmp_srcs
                               __tmp_hdrs
                               ${ONNX_ROOT}
                               ""
                               onnx/onnx.in.proto)
list(APPEND ONNX_PROTO_SRCS ${__tmp_srcs})
list(APPEND ONNX_PROTO_HDRS ${__tmp_hdrs})

relative_protobuf_generate_cpp(gen_onnx_operators_proto
                               __tmp_srcs
                               __tmp_hdrs
                               ${ONNX_ROOT}
                               gen_onnx_proto
                               onnx/onnx-operators.in.proto)
list(APPEND ONNX_PROTO_SRCS ${__tmp_srcs})
list(APPEND ONNX_PROTO_HDRS ${__tmp_hdrs})

relative_protobuf_generate_cpp(gen_onnx_data_proto
                               __tmp_srcs
                               __tmp_hdrs
                               ${ONNX_ROOT}
                               gen_onnx_proto
                               onnx/onnx-data.in.proto)
list(APPEND ONNX_PROTO_SRCS ${__tmp_srcs})
list(APPEND ONNX_PROTO_HDRS ${__tmp_hdrs})

file(GLOB_RECURSE __tmp_srcs "${ONNX_ROOT}/onnx/*.h" "${ONNX_ROOT}/onnx/*.cc")
file(GLOB_RECURSE onnx_gtests_src "${ONNX_ROOT}/onnx/test/cpp/*.h"
    "${ONNX_ROOT}/onnx/test/cpp/*.cc"
    "${ONNX_ROOT}/onnx/backend/test/cpp/*.cc"
    "${ONNX_ROOT}/onnx/backend/test/cpp/*.h")
list(REMOVE_ITEM __tmp_srcs "${ONNX_ROOT}/onnx/cpp2py_export.cc")
list(REMOVE_ITEM __tmp_srcs ${onnx_gtests_src})
list(APPEND ONNX_SRCS ${__tmp_srcs})

add_library(onnx_proto ${ONNX_PROTO_SRCS} ${ONNX_PROTO_HDRS})
add_dependencies(onnx_proto gen_onnx_operators_proto gen_onnx_data_proto)
target_include_directories(onnx_proto PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
  $<INSTALL_INTERFACE:include>
  $<BUILD_INTERFACE:${PROTOBUF_INCLUDE_DIRS}>)

if (MSVC)
  if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
    set(EXTRA_FLAGS
          "-Wno-implicit-function-declaration"
          "-Wno-undefined-inline"
          "-Wno-incompatible-pointer-types"
          "-Wno-dllexport-explicit-instantiation-decl"
          "-Wno-microsoft-unqualified-friend"
          "-Wno-absolute-value"
          "-Wno-unused-variable"
          "-Wno-unknown-argument"
          "-Wno-writable-strings"
          "-Qunused-arguments")
  else()
    set(EXTRA_FLAGS "")
  endif()
  if (BUILD_SHARED_LIBS OR ONNX_BUILD_MAIN_LIB)
    set(ONNX_API_DEFINE "-DONNX_API=__declspec(dllexport)")
  else()
    set(ONNX_API_DEFINE "-DONNX_API=")
  endif()
else()
  # On non-Windows, hide all symbols we don't need
  set(ONNX_API_DEFINE "-DONNX_API=__attribute__\(\(__visibility__\(\"default\"\)\)\)")
  set_target_properties(onnx_proto PROPERTIES CXX_VISIBILITY_PRESET hidden)
  set_target_properties(onnx_proto PROPERTIES VISIBILITY_INLINES_HIDDEN 1)
endif()
target_compile_definitions(onnx_proto PRIVATE ${ONNX_API_DEFINE})

if(ONNX_USE_LITE_PROTO)
  if(TARGET protobuf::libprotobuf-lite)
    target_link_libraries(onnx_proto PUBLIC protobuf::libprotobuf-lite)
  else()
    target_link_libraries(onnx_proto PUBLIC ${PROTOBUF_LITE_LIBRARIES})
  endif()
else()
  if(TARGET protobuf::libprotobuf)
    target_link_libraries(onnx_proto PUBLIC protobuf::libprotobuf)
  else()
    target_link_libraries(onnx_proto PUBLIC ${PROTOBUF_LIBRARIES})
  endif()
endif()
add_onnx_global_defines(onnx_proto)

if(CMAKE_SYSTEM_NAME STREQUAL "AIX")
  # whole-archive linker option not available on AIX.
  # So, create a object library
  add_library(onnx OBJECT ${ONNX_SRCS})
else()
  add_library(onnx ${ONNX_SRCS})
endif()

target_include_directories(onnx PUBLIC
  $<BUILD_INTERFACE:${ONNX_ROOT}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
  $<INSTALL_INTERFACE:include>)
target_link_libraries(onnx PUBLIC onnx_proto)
add_onnx_global_defines(onnx)

if(BUILD_ONNX_PYTHON)
  if("${PY_EXT_SUFFIX}" STREQUAL "")
    if(MSVC)
      set(PY_EXT_SUFFIX ".pyd")
    else()
      set(PY_EXT_SUFFIX ".so")
    endif()
  endif()

  add_library(onnx_cpp2py_export MODULE "${ONNX_ROOT}/onnx/cpp2py_export.cc")
  set_target_properties(onnx_cpp2py_export PROPERTIES PREFIX "")
  set_target_properties(onnx_cpp2py_export
                        PROPERTIES COMPILE_FLAGS "-fvisibility=hidden")
  set_target_properties(onnx_cpp2py_export PROPERTIES SUFFIX ${PY_EXT_SUFFIX})
  set_target_properties(onnx_cpp2py_export
                        PROPERTIES LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
  target_include_directories(onnx_cpp2py_export PRIVATE
                             $<BUILD_INTERFACE:${ONNX_ROOT}>
                             $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
                             $<INSTALL_INTERFACE:include>)

  # pybind11 is a header only lib
  find_package(pybind11 2.2 CONFIG)
  if(NOT pybind11_FOUND)
    if(EXISTS "${ONNX_ROOT}/third_party/pybind11/include/pybind11/pybind11.h")
      add_subdirectory("${ONNX_ROOT}/third_party/pybind11")
    else()
      message(FATAL_ERROR "cannot find pybind")
    endif()
  endif()

  target_include_directories(onnx_cpp2py_export PUBLIC
    "${pybind11_INCLUDE_DIRS}"
    "${PYTHON_INCLUDE_DIRS}")

  if(APPLE)
    set_target_properties(onnx_cpp2py_export
                          PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
    # Only put double quotes around $<TARGET_FILE:onnx> for Mac
    # Other platforms like Windows and Ubuntu originally work fine without double quotes
    target_link_libraries(onnx_cpp2py_export
                          PRIVATE -Wl,-force_load,"$<TARGET_FILE:onnx>")
  elseif(MSVC)
    # In MSVC, we will add whole archive in default
    target_link_libraries(onnx_cpp2py_export
                          PRIVATE -WHOLEARCHIVE:$<TARGET_FILE:onnx>)
  elseif(CMAKE_SYSTEM_NAME STREQUAL "AIX")
    # whole-archive linker option not available on AIX
    target_sources(onnx_cpp2py_export
                          PRIVATE $<TARGET_OBJECTS:onnx>)
  else()
    # Assume everything else is like gcc
    target_link_libraries(onnx_cpp2py_export
                          PRIVATE "-Wl,--whole-archive" $<TARGET_FILE:onnx>
                                  "-Wl,--no-whole-archive")
    set_target_properties(onnx_cpp2py_export
                          PROPERTIES LINK_FLAGS "-Wl,--exclude-libs,ALL")
  endif()

  target_link_libraries(onnx_cpp2py_export PRIVATE onnx)

  if(MSVC)
    target_link_libraries(onnx_cpp2py_export PRIVATE ${PYTHON_LIBRARIES})
    target_compile_options(onnx_cpp2py_export
                           PRIVATE /MP
                                   /wd4244 # 'argument': conversion from 'google::
                                           # protobuf::uint64' to 'int', possible
                                           # loss of data
                                   /wd4267 # Conversion from 'size_t' to 'int',
                                           # possible loss of data
                                   /wd4996 # The second parameter is ignored.
                                   ${EXTRA_FLAGS})
    if(ONNX_USE_PROTOBUF_SHARED_LIBS)
      target_compile_options(onnx_cpp2py_export
                             PRIVATE /wd4251 # 'identifier' : class 'type1' needs to
                                             # have dll-interface to be used by
                                             # clients of class 'type2'
                            )
    endif()
    add_msvc_runtime_flag(onnx_cpp2py_export)
    add_onnx_global_defines(onnx_cpp2py_export)
  endif()
endif()

if(ONNX_BUILD_BENCHMARKS)
  if(NOT TARGET benchmark)
    # We will not need to test benchmark lib itself.
    set(BENCHMARK_ENABLE_TESTING OFF
        CACHE BOOL "Disable benchmark testing as we don't need it.")
    # We will not need to install benchmark since we link it statically.
    set(BENCHMARK_ENABLE_INSTALL OFF
        CACHE BOOL
              "Disable benchmark install to avoid overwriting vendor install.")
    add_subdirectory(${PROJECT_SOURCE_DIR}/third_party/benchmark)
  endif()

  add_executable(protobuf-bench tools/protobuf-bench.cc)
  target_include_directories(protobuf-bench PUBLIC
    $<BUILD_INTERFACE:${ONNX_ROOT}>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
    $<INSTALL_INTERFACE:include>
    $<BUILD_INTERFACE:${PROTOBUF_INCLUDE_DIRS}>)
  target_link_libraries(protobuf-bench onnx_proto benchmark)
endif()

# Export include directories
set(ONNX_INCLUDE_DIRS "${ONNX_ROOT}" "${CMAKE_CURRENT_BINARY_DIR}")
get_directory_property(hasParent PARENT_DIRECTORY)
if(hasParent)
  set(ONNX_INCLUDE_DIRS ${ONNX_INCLUDE_DIRS} PARENT_SCOPE)
endif()

if(MSVC)
  target_compile_options(onnx_proto
                         PRIVATE /MP
                                 /wd4244 #'argument': conversion from 'google::
                                         #protobuf::uint64' to 'int', possible
                                         # loss of data
                                 /wd4267 # Conversion from 'size_t' to 'int',
                                         # possible loss of data
                                 ${EXTRA_FLAGS})
  target_compile_options(onnx
                         PRIVATE /MP
                                 /wd4244 # 'argument': conversion from 'google::
                                         # protobuf::uint64' to 'int', possible
                                         # loss of data
                                 /wd4267 # Conversion from 'size_t' to 'int',
                                         # possible loss of data
                                 /wd4996 # The second parameter is ignored.
                                 ${EXTRA_FLAGS})
  if(ONNX_USE_PROTOBUF_SHARED_LIBS)
      target_compile_options(onnx_proto
                             PRIVATE /wd4251 # 'identifier' : class 'type1' needs to
                                             # have dll-interface to be used by
                                             # clients of class 'type2'
                            )
      target_compile_options(onnx
                             PRIVATE /wd4251 # 'identifier' : class 'type1' needs to
                                             # have dll-interface to be used by
                                             # clients of class 'type2'
                            )
  endif()
  add_msvc_runtime_flag(onnx_proto)
  add_msvc_runtime_flag(onnx)
  set(onnx_static_library_flags
      -IGNORE:4221 # LNK4221: This object file does not define any previously
                   # undefined public symbols, so it will not be used by any
                   # link operation that consumes this library
      )
  set_target_properties(onnx
                        PROPERTIES STATIC_LIBRARY_FLAGS
                                   "${onnx_static_library_flags}")
elseif(APPLE)

else()
  target_compile_options(onnx PRIVATE -Wall -Wextra)
endif()

if(APPLE)
  set_target_properties(onnx PROPERTIES LINK_FLAGS "-undefined dynamic_lookup")
endif()

install(DIRECTORY ${ONNX_ROOT}/onnx
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        FILES_MATCHING
        PATTERN "*.h"
        PATTERN "backend/test/case" EXCLUDE
        PATTERN "backend/test/data" EXCLUDE)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/onnx
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        FILES_MATCHING
        PATTERN "*.h")

configure_file(
  ${PROJECT_SOURCE_DIR}/cmake/ONNXConfigVersion.cmake.in
  ${PROJECT_BINARY_DIR}/ONNXConfigVersion.cmake
  @ONLY)
configure_file(
  ${PROJECT_SOURCE_DIR}/cmake/ONNXConfig.cmake.in
  ${PROJECT_BINARY_DIR}/ONNXConfig.cmake
  @ONLY)
install(FILES
  ${PROJECT_BINARY_DIR}/ONNXConfigVersion.cmake
  ${PROJECT_BINARY_DIR}/ONNXConfig.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/ONNX
  COMPONENT dev)
install(EXPORT ONNXTargets DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/ONNX")
install(TARGETS
  onnx onnx_proto
  EXPORT ONNXTargets DESTINATION ${CMAKE_INSTALL_LIBDIR})

if(ONNX_BUILD_TESTS)
  include(${ONNX_ROOT}/cmake/unittest.cmake)
endif()

# Support older g++ to use PRId64
add_definitions(-D__STDC_FORMAT_MACROS)

include(cmake/summary.cmake)
onnx_print_configuration_summary()
