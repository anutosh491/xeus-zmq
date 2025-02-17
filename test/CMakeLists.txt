############################################################################
# Copyright (c) 2016, Sylvain Corlay, Johan Mabille, Martin Renou          #
# Copyright (c) 2016, QuantStack                                           #
#                                                                          #
# Distributed under the terms of the BSD 3-Clause License.                 #
#                                                                          #
# The full license is in the file LICENSE, distributed with this software. #
############################################################################

# Unit tests
# ==========

cmake_minimum_required(VERSION 3.1)

if (CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    project(xeus-zmq-test)

    enable_testing()

    find_package(xeus-zmq REQUIRED CONFIG)
    find_package(nlohmann_json QUIET CONFIG)

    set(XEUS_ZMQ_TEST_DIR ${CMAKE_CURRENT_LIST_DIR})
endif ()

message(STATUS "Forcing tests build type to Release")
set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)

include(CheckCXXCompilerFlag)

string(TOUPPER "${CMAKE_BUILD_TYPE}" U_CMAKE_BUILD_TYPE)

if(nlohmann_json_FOUND)
  add_definitions(-DHAVE_NLOHMANN_JSON)
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES Clang OR CMAKE_CXX_COMPILER_ID MATCHES GNU OR CMAKE_CXX_COMPILER_ID MATCHES Intel)
    add_compile_options(-Wunused-parameter -Wextra -Wreorder -Wconversion -Wsign-conversion)

    CHECK_CXX_COMPILER_FLAG(-march=native HAS_MARCH_NATIVE)
    if (HAS_MARCH_NATIVE)
        add_compile_options(-march=native)
    endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES MSVC)
    add_compile_options(/EHsc /MP /bigobj)
    set(CMAKE_EXE_LINKER_FLAGS /MANIFEST:NO)
endif()


find_package(doctest REQUIRED)
find_package(Threads)

find_program(PYTEST  
    NAMES pytest-3 pytest py.test-3 py.test REQUIRED)

if(nlohmann_json_FOUND)
    # Version up to 3.1.2 export the target `nlohmann_json`
    if(TARGET nlohmann_json)
      set(nlohmann_json_TARGET nlohmann_json)
    # Newer versions export the namespaced target `nlohmann_json::nlohmann_json`
    elseif(TARGET nlohmann_json::nlohmann_json)
      set(nlohmann_json_TARGET nlohmann_json::nlohmann_json)
    endif()
endif()

if (TARGET xeus-zmq)
    set(xeus-zmq_TARGET xeus-zmq)
    message("Found xeus-zmq shared library.")
elseif (TARGET xeus-zmq-static)
    set(xeus-zmq_TARGET xeus-zmq-static)
    message("Found xeus-zmq static library.")
endif ()

# Test_kernel tests
# =================

set(TEST_KERNEL_SOURCES
    xmock_interpreter.cpp
    xmock_interpreter.hpp
    main.cpp)

configure_file(
    "${XEUS_ZMQ_TEST_DIR}/test_kernel/kernel.json.in"
    "kernels/test_kernel/kernel.json"
)

add_executable(test_kernel ${TEST_KERNEL_SOURCES})
target_link_libraries(test_kernel PRIVATE ${xeus-zmq_TARGET} Threads::Threads)
target_compile_features(test_kernel PRIVATE cxx_std_11)

set(CONNECTION_FILE ${CMAKE_CURRENT_SOURCE_DIR}/connection.json)

configure_file(
    "${XEUS_ZMQ_TEST_DIR}/test_kernel.py"
    "${CMAKE_CURRENT_BINARY_DIR}/"
    COPYONLY)

add_test(NAME test_kernel
    COMMAND ${PYTEST} test_kernel.py)
set_tests_properties(test_kernel
    PROPERTIES
    ENVIRONMENT "JUPYTER_PATH=${CMAKE_CURRENT_BINARY_DIR}")

# Test_kernel_control tests
# =========================

set(TEST_KERNEL_SPLIT_SOURCES
    xmock_interpreter.cpp
    xmock_interpreter.hpp
    main_control.cpp)

configure_file(
    "${XEUS_ZMQ_TEST_DIR}/test_kernel_control/kernel.json.in"
    "kernels/test_kernel_control/kernel.json"
)

add_executable(test_kernel_control ${TEST_KERNEL_SPLIT_SOURCES})
target_link_libraries(test_kernel_control PRIVATE ${xeus-zmq_TARGET} Threads::Threads)
target_compile_features(test_kernel_control PRIVATE cxx_std_11)

set(CONNECTION_FILE ${CMAKE_CURRENT_SOURCE_DIR}/connection.json)

configure_file(
    "${XEUS_ZMQ_TEST_DIR}/test_kernel_control.py"
    "${CMAKE_CURRENT_BINARY_DIR}/"
    COPYONLY)

add_test(NAME test_kernel_control
    COMMAND ${PYTEST} test_kernel_control.py)
set_tests_properties(test_kernel_control
    PROPERTIES
    ENVIRONMENT "JUPYTER_PATH=${CMAKE_CURRENT_BINARY_DIR}")

# Test_kernel_shell tests
# =======================

set(TEST_KERNEL_SPLIT_SOURCES
    xmock_interpreter.cpp
    xmock_interpreter.hpp
    main_shell.cpp)

configure_file(
    "${XEUS_ZMQ_TEST_DIR}/test_kernel_shell/kernel.json.in"
    "kernels/test_kernel_shell/kernel.json"
)

add_executable(test_kernel_shell ${TEST_KERNEL_SPLIT_SOURCES})
target_link_libraries(test_kernel_shell PRIVATE ${xeus-zmq_TARGET} Threads::Threads)
target_compile_features(test_kernel_shell PRIVATE cxx_std_11)

set(CONNECTION_FILE ${CMAKE_CURRENT_SOURCE_DIR}/connection.json)

configure_file(
    "${XEUS_ZMQ_TEST_DIR}/test_kernel_shell.py"
    "${CMAKE_CURRENT_BINARY_DIR}/"
    COPYONLY)

add_test(NAME test_kernel_shell
    COMMAND ${PYTEST} test_kernel_shell.py)
set_tests_properties(test_kernel_shell
    PROPERTIES
    ENVIRONMENT "JUPYTER_PATH=${CMAKE_CURRENT_BINARY_DIR}")

if (UNIX)

    # Test_kernel_ipc
    # ===============

    set(TEST_KERNEL_IPC_SOURCES
        xmock_interpreter.cpp
        xmock_interpreter.hpp
        main_ipc.cpp)

    add_executable(test_kernel_ipc ${TEST_KERNEL_IPC_SOURCES})
    target_link_libraries(test_kernel_ipc PRIVATE ${xeus-zmq_TARGET} Threads::Threads)
    target_compile_features(test_kernel_ipc PRIVATE cxx_std_11)

    # Test_client_ipc
    # ===============

    set(TEST_CLIENT_IPC_SOURCES
        client_ipc.cpp)

    add_executable(test_client_ipc ${TEST_CLIENT_IPC_SOURCES})
    target_link_libraries(test_client_ipc PRIVATE ${xeus-zmq_TARGET} Threads::Threads)
    target_compile_features(test_client_ipc PRIVATE cxx_std_11)

endif (UNIX)
