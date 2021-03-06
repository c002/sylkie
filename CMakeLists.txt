# Copyright (c) 2017 Daniel L. Robertson
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.

cmake_minimum_required(VERSION 3.5)
project(
    sylkie
    VERSION 0.0.4
    LANGUAGES C CXX
    )

set(LIB_DIR "${PROJECT_SOURCE_DIR}/lib")
set(EXE_DIR "${PROJECT_SOURCE_DIR}/src")
set(INCLUDE_DIR "${PROJECT_SOURCE_DIR}/include")
set(TEST_DIR "${PROJECT_SOURCE_DIR}/tests")
set(CMAKE_EXTRA "${PROJECT_SOURCE_DIR}/CMake")

include(CheckIncludeFile)
include(ExternalProject)

# Additional options

# Enable features
set(ENABLE_JSON AUTO CACHE STRING "Enable json support")
set_property(CACHE ENABLE_JSON PROPERTY STRINGS AUTO ON OFF)
set(ENABLE_SECCOMP AUTO CACHE STRING "Enable seccomp support")
set_property(CACHE ENABLE_SECCOMP PROPERTY STRINGS AUTO ON OFF)
set(ENABLE_SETUID AUTO CACHE STRING "Set the setuid bit on the executable")
set_property(CACHE ENABLE_SETUID PROPERTY STRINGS AUTO ON OFF)
set(ENABLE_STATIC CACHE STRING "Statically link the executable")
set_property(CACHE ENABLE_STATIC PROPERTY STRINGS ON OFF)

# Build extras
set(BUILD_TESTS OFF CACHE STRING "Build tests")
set_property(CACHE BUILD_TESTS PROPERTY STRINGS ON OFF)
set(BUILD_DOCS OFF CACHE STRING "Build documentation")
set_property(CACHE BUILD_DOCS PROPERTY STRINGS ON OFF)

# Deprecated options
option(JSON "JSON" OFF)
option(SECCOMP "SECCOMP" OFF)
option(ENABLE_TESTS "ENABLE_TESTS" OFF)

if(JSON)
    set(ENABLE_JSON "ON")
    message(WARNING "JSON option has been deprecated. Use ENABLE_JSON.")
endif(JSON)
if(SECCOMP)
    set(ENABLE_SECCOMP "ON")
    message(WARNING "SECCOMP option has been deprecated. Use ENABLE_SECCOMP.")
endif(SECCOMP)
if(ENABLE_TESTS)
    set(BUILD_TESTS "ON")
    message(WARNING "ENABLE_TESTS option has been deprecated. Use BUILD_TESTS.")
endif(ENABLE_TESTS)

# Check OS

message("-- Check for OS support")

if(NOT UNIX)
    message(FATAL_ERROR "This project is not yet supported on non-unix systems")
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set(BUILD_OS "linux")
    set(BUILD_LINUX 1)
else(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    message(FATAL_ERROR "This project is not yet supported on ${CMAKE_SYSTEM_NAME}")
endif(${CMAKE_SYSTEM_NAME} MATCHES "Linux")

# Check kernel version

message("-- Check kernel version")

set(KERNEL_VERSION_STRING ${CMAKE_SYSTEM_VERSION})

string(REGEX MATCH "([0-9]+)\\.([0-9]+)[.-]([0-9]+)"
    KERNEL_MATCHED "${KERNEL_VERSION_STRING}")

if(NOT KERNEL_MATCHED)
    message(FATAL_ERROR "${CMAKE_SYSTEM_VERSION} did not match kernel regex")
endif(NOT KERNEL_MATCHED)

set(KERNEL_MAJOR_VERSION ${CMAKE_MATCH_1})
set(KERNEL_MINOR_VERSION ${CMAKE_MATCH_2})
set(KERNEL_PATCH_VERSION ${CMAKE_MATCH_3})

set(KERNEL_VERSION "${CMAKE_MATCH_1}.${CMAKE_MATCH_2}.${CMAKE_MATCH_4}")

if(KERNEL_VERSION VERSION_LESS 3.0)
    message(FATAL_ERROR "Bad kernel version")
else(KERNEL_VERSION VERSION_LESS 3.0)
    message("-- Check kernel version -- works")
endif(KERNEL_VERSION VERSION_LESS 3.0)

# Check for headers in CMake/headers.txt

function(header_check filename)
    if(EXISTS ${filename})
        file(STRINGS ${filename} HEADERS REGEX "[A-z\/]+\\.h")
        foreach(f ${HEADERS})
            check_include_file(${f} HAVE_HEADER_${f})
            if(NOT HAVE_HEADER_${f})
                message(FATAL_ERROR "Could not find ${f}")
            endif(NOT HAVE_HEADER_${f})
        endforeach(f)
    endif(EXISTS ${filename})
endfunction(header_check)

# Check for default headers
header_check(${CMAKE_EXTRA}/headers.txt)

# Check for OS specific headers
header_check(${CMAKE_EXTRA}/${BUILD_OS}/headers.txt)

# Basic build

include_directories(${INCLUDE_DIR})

file(GLOB INCLUDE_FILES ${INCLUDE_DIR}/*.h)

file(GLOB LIB_SRCS ${LIB_DIR}/*.c)

file(GLOB EXE_SRCS ${EXE_DIR}/*.c)
list(REMOVE_ITEM EXE_SRCS "${EXE_DIR}/main.c")

# CFLAGS

set(DEFAULT_DEBUG "-g -O0")
set(DEV_WARNINGS "-Wall -Wshadow -Wstrict-overflow -fno-strict-aliasing")
set(CMAKE_C_FLAGS_DEBUG
    "${CMAKE_C_FLAGS_DEBUG} ${DEFAULT_DEBUG} ${DEV_WARNINGS}")
set(CMAKE_CXX_FLAGS_DEBUG
    "${CMAKE_CXX_FLAGS_DEBUG} -std=c++11 ${DEFAULT_DEBUG} ${DEV_WARNINGS}")
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O2 -Wall")
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -std=c++11 -O2 -Wall")

# Add custom build type

# Add new build types
set(CMAKE_C_FLAGS_ASAN
    "${DEFAULT_DEBUG} ${DEV_WARNINGS} -fsanitize=address -fno-omit-frame-pointer"
    CACHE STRING "Flags used by the C compiler during address sanitizer builds."
    FORCE )
set(CMAKE_CXX_FLAGS_ASAN
    "${DEFAULT_DEBUG} ${DEV_WARNINGS} -std=c++11 -fsanitize=address -fno-omit-frame-pointer"
    CACHE STRING "Flags used by the C++ compiler during address sanitizer builds."
    FORCE )
set(CMAKE_EXE_LINKER_FLAGS_ASAN
    ""
    CACHE STRING "Flags used for linking binaries during address sanitizer builds."
    FORCE )
set(CMAKE_SHARED_LINKER_FLAGS_ASAN
    ""
    CACHE STRING "Flags used by the shared libraries linker during address
    sanitizer builds."
    FORCE )
mark_as_advanced(
    CMAKE_CXX_FLAGS_ASAN
    CMAKE_C_FLAGS_ASAN
    CMAKE_EXE_LINKER_FLAGS_ASAN
    CMAKE_SHARED_LINKER_FLAGS_ASAN
    )

# Outputs

add_library(${PROJECT_NAME}-shared SHARED ${LIB_SRCS})
set_target_properties(${PROJECT_NAME}-shared PROPERTIES
    OUTPUT_NAME ${PROJECT_NAME})
set_target_properties(${PROJECT_NAME}-shared PROPERTIES
    SOVERSION ${PROJECT_VERSION_MAJOR})
set_property(TARGET ${PROJECT_NAME}-shared PROPERTY C_STANDARD 11)

add_library(${PROJECT_NAME}-static STATIC ${LIB_SRCS})
set_target_properties(${PROJECT_NAME}-static PROPERTIES
    OUTPUT_NAME ${PROJECT_NAME})
set_property(TARGET ${PROJECT_NAME}-static PROPERTY C_STANDARD 11)

add_executable(${PROJECT_NAME}-bin ${EXE_SRCS} ${EXE_DIR}/main.c)
target_include_directories(${PROJECT_NAME}-bin PRIVATE ${EXE_DIR}/include)
set_target_properties(${PROJECT_NAME}-bin PROPERTIES
    OUTPUT_NAME ${PROJECT_NAME})
set_property(TARGET ${PROJECT_NAME}-bin PROPERTY C_STANDARD 11)
if(ENABLE_STATIC)
    target_link_libraries(${PROJECT_NAME}-bin PRIVATE
        ${PROJECT_NAME}-static)
else(ENABLE_STATIC)
    target_link_libraries(${PROJECT_NAME}-bin PRIVATE
        ${PROJECT_NAME}-shared)
endif(ENABLE_STATIC)

# Dependencies

if(ENABLE_JSON OR ENABLE_JSON STREQUAL "AUTO")
    check_include_file(json-c/json.h JSON_HEADER_FOUND)
    find_library(JSON_LIB json-c)
    if(JSON_LIB AND JSON_HEADER_FOUND)
        target_link_libraries(${PROJECT_NAME}-bin PRIVATE json-c)
    else(JSON_LIB AND JSON_HEADER_FOUND)
        message(FATAL_ERROR "Could not find json-c library. Either install "
            "json-c or run with -DJSON=OFF to build without json support")
    endif(JSON_LIB AND JSON_HEADER_FOUND)
    set(BUILD_JSON 1)
endif(ENABLE_JSON OR ENABLE_JSON STREQUAL "AUTO")

if(ENABLE_SECCOMP OR ENABLE_SECCOMP STREQUAL "AUTO")
    check_include_file(seccomp.h SECCOMP_HEADER_FOUND)
    check_include_file(sys/prctl.h PRCTL_HEADER_FOUND)
    find_library(SECCOMP_LIB seccomp)
    if(SECCOMP_LIB AND SECCOMP_HEADER_FOUND AND PRCTL_HEADER_FOUND)
        target_link_libraries(${PROJECT_NAME}-bin PRIVATE seccomp)
    else(SECCOMP_LIB AND SECCOMP_HEADER_FOUND AND PRCTL_HEADER_FOUND)
        message(FATAL_ERROR "Could not find seccomp library. Either install "
            "libseccomp or run with -DSECCOMP=OFF to build without seccomp support")
    endif(SECCOMP_LIB AND SECCOMP_HEADER_FOUND AND PRCTL_HEADER_FOUND)
    set(BUILD_SECCOMP 1)
endif(ENABLE_SECCOMP OR ENABLE_SECCOMP STREQUAL "AUTO")

# Configure sylkie_config.h

configure_file(${INCLUDE_DIR}/config.in ${INCLUDE_DIR}/${PROJECT_NAME}_config.h)

# Lints

add_custom_target(clang-tidy COMMAND clang-tidy ${EXE_DIR}/*.c ${LIB_DIR}/*.c
    -- -std=c11 -I${INCLUDE_DIR} -I${EXE_DIR}/include)

add_custom_target(clang-format COMMAND clang-format -i -style=file
    ${EXE_DIR}/*.c ${EXE_DIR}/include/*.h ${LIB_DIR}/*.c
    ${TEST_DIR}/*.cpp ${INCLUDE_DIR}/*.h)

# Docs

if(BUILD_DOCS)
    find_package(Doxygen)
    if(DOXYGEN_FOUND)
        set(DOC_FILES "")
        foreach(str ${LIB_SRCS})
            set(DOC_FILES "${DOC_FILES} ${str}")
        endforeach(str)
        foreach(str ${INCLUDE_FILES})
            set(DOC_FILES "${DOC_FILES} ${str}")
        endforeach(str)
        configure_file(${PROJECT_SOURCE_DIR}/Doxyfile.in
            ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)
        add_custom_target(doc
            ${DOXYGEN_EXECUTABLE}
            ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
            WORKING DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
            )
    else(DOXYGEN_FOUND)
        message(FATAL_ERROR "Must have doxygen install to generate docs")
    endif(DOXYGEN_FOUND)
endif(BUILD_DOCS)

# Tests

if(BUILD_TESTS)
    enable_testing()

    include(CTest)

    find_package(GTest)

    file(GLOB TEST_SRCS ${TEST_DIR}/test_*.cpp)

    add_executable(${PROJECT_NAME}_test ${TEST_DIR}/runner.cpp ${TEST_SRCS} ${EXE_SRCS}
        ${LIB_SRCS})
    set_target_properties(${PROJECT_NAME}_test PROPERTIES LINKER_LANGUAGE CXX)
    target_include_directories(${PROJECT_NAME}_test PRIVATE ${EXE_DIR}/include)
    if (ENABLE_JSON)
        target_link_libraries(${PROJECT_NAME}_test json-c)
    endif(ENABLE_JSON)
    if(ENABLE_SECCOMP)
        target_link_libraries(${PROJECT_NAME}_test seccomp)
    endif(ENABLE_SECCOMP)

    if(GTEST_FOUND)
        include_directories(${GTEST_INCLUDE_DIRS})
        target_link_libraries(${PROJECT_NAME}_test gtest)
    else(GTEST_FOUND)
        # From the gtest README.md

        # Download and unpack googletest at configure time
        set(GTEST_VERSION "1.8.0")
        configure_file(CMake/GTest.txt.in googletest-download/CMakeLists.txt)
        execute_process(COMMAND ${CMAKE_COMMAND} -G "${CMAKE_GENERATOR}" .
          RESULT_VARIABLE result
          WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/googletest-download )
        if(result)
          message(FATAL_ERROR "CMake step for googletest failed: ${result}")
        endif()
        execute_process(COMMAND ${CMAKE_COMMAND} --build .
          RESULT_VARIABLE result
          WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/googletest-download )
        if(result)
          message(FATAL_ERROR "Build step for googletest failed: ${result}")
        endif()

        # Prevent overriding the parent project's compiler/linker
        # settings on Windows
        set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)

        # Add googletest directly to our build. This defines
        # the gtest and gtest_main targets.
        add_subdirectory(${CMAKE_BINARY_DIR}/googletest-src
                         ${CMAKE_BINARY_DIR}/googletest-build)

        # Link against gtest
        target_link_libraries(${PROJECT_NAME}_test gtest_main)
    endif(GTEST_FOUND)


    foreach(test ${TEST_SRCS})
        get_filename_component(TEST_NAME ${test} NAME_WE)
        add_test(${PROJECT_NAME}_test ${PROJECT_NAME}_test ${TEST_NAME})
    endforeach(test)

    add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND})
    set_property(TARGET ${PROJECT_NAME}_test PROPERTY C_STANDARD 11)
    set_property(TARGET ${PROJECT_NAME}_test PROPERTY CXX_STANDARD 11)
endif(BUILD_TESTS)

# Install
set(DEFAULT_EXEC_PERMISSIONS
    OWNER_WRITE OWNER_READ OWNER_EXECUTE
    GROUP_READ GROUP_EXECUTE
    WORLD_READ WORLD_EXECUTE)

set(DEFAULT_READ_PERMISSIONS
    OWNER_WRITE OWNER_READ
    GROUP_READ
    WORLD_READ)

install(
    TARGETS ${PROJECT_NAME}-shared ${PROJECT_NAME}-static
    LIBRARY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
    ARCHIVE DESTINATION ${CMAKE_INSTALL_PREFIX}/lib
    )

if(ENABLE_SETUID OR ENABLE_SETUID STREQUAL "AUTO")
    install(
        TARGETS ${PROJECT_NAME}-bin
        RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
        PERMISSIONS ${DEFAULT_EXEC_PERMISSIONS} SETUID
        )
else(ENABLE_SETUID OR ENABLE_SETUID STREQUAL "AUTO")
    install(
        TARGETS ${PROJECT_NAME}-bin
        RUNTIME DESTINATION ${CMAKE_INSTALL_PREFIX}/bin
        PERMISSIONS ${DEFAULT_EXEC_PERMISSIONS}
        )
endif(ENABLE_SETUID OR ENABLE_SETUID STREQUAL "AUTO")

install(
    FILES ${INCLUDE_FILES}
    DESTINATION ${CMAKE_INSTALL_PREFIX}/include/${PROJECT_NAME}
    PERMISSIONS ${DEFAULT_READ_PERMISSIONS}
    )
