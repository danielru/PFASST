cmake_minimum_required(VERSION 2.8)
project(pfasst)

list(APPEND CMAKE_MODULE_PATH ${pfasst_SOURCE_DIR}/cmake)
include(cmake/utility_functions.cmake)
include(CheckCXXCompilerFlag)
include(ExternalProject)

# Set default ExternalProject root directory
set_directory_properties(PROPERTIES EP_PREFIX ${CMAKE_BINARY_DIR}/3rdparty)

option(pfasst_DISABLE_LIBCXX "Disable use of LLVM's libc++ when compiling with Clang." ON )
option(pfasst_BUILD_EXAMPLES "Build example programs."                                 ON )
option(pfasst_BUILD_TESTS    "Build test suite for PFASST."                            ON )
option(pfasst_WITH_MPI       "Build with MPI enabled."                                 OFF)
option(pfasst_WITH_GCC_PROF  "Enable excessive debugging & profiling output with GCC." OFF)

# Set output directories
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${pfasst_SOURCE_DIR}/dist/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${pfasst_SOURCE_DIR}/dist/lib")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${pfasst_SOURCE_DIR}/dist/lib")

if(${pfasst_WITH_MPI})
    find_package(MPI REQUIRED)
    message(STATUS "Using MPI C++ Compiler: ${MPI_CXX_COMPILER}")
endif()

# Check for C++11 support
if(${CMAKE_CXX_COMPILER_ID} MATCHES GNU)
    check_cxx_compiler_flag(-std=c++11 HAVE_STD11)
    if(HAVE_STD11)
        add_to_string_list("${CMAKE_CXX_FLAGS}" CMAKE_CXX_FLAGS "-std=c++11")
    else()
        message(FATAL_ERROR "No advanced standard C++ support of your GCC (-std=c++11 not defined).")
    endif()
    if(pfasst_WITH_GCC_PROF)
        add_to_string_list("${CMAKE_CXX_FLAGS}" CMAKE_CXX_FLAGS "-ggdb3 -pg")
    endif(pfasst_WITH_GCC_PROF)
elseif(${CMAKE_CXX_COMPILER_ID} MATCHES Clang)
    check_cxx_compiler_flag(-std=c++11 HAVE_STD11)
    if(HAVE_STD11)
        if(pfasst_DISABLE_LIBCXX AND NOT APPLE)
            add_to_string_list("${CMAKE_CXX_FLAGS}" CMAKE_CXX_FLAGS "-std=c++11")
            add_to_string_list("${CMAKE_CXX_LINK_FLAGS}" CMAKE_CXX_LINK_FLAGS "-std=c++11")
        else()
            if(APPLE)
                message(WARNING "You are on an Apple system. libc++ is forcly enabled.")
            endif(APPLE)
            add_to_string_list("${CMAKE_CXX_FLAGS}" CMAKE_CXX_FLAGS "-std=c++11" "-stdlib=libc++")
            add_to_string_list("${CMAKE_CXX_LINK_FLAGS}" CMAKE_CXX_LINK_FLAGS "-std=c++11" "-stdlib=libc++")
        endif()
    else()
        message(FATAL_ERROR "No C++11 support for Clang version. Please upgrade Clang to a version supporting C++11.")
    endif()
elseif(${CMAKE_CXX_COMPILER_ID} MATCHES XL)
    # NOTE: This branch is not tested yet (in theory it should work)
    message(WARNING "IMB XL C/C++ support is experimental and not yet tested.")
    check_cxx_compiler_flag(-qlanglvl=extended0x HAVE_STD11)
    if(HAVE_STD11)
        add_to_string_list("${CMAKE_CXX_FLAGS}" CMAKE_CXX_FLAGS "-qlanglvl=extended0x -qwarn0x")
    else()
        message(FATAL_ERROR "No advanced standard C++ support of your IBM XL C/C++ compiler (-qlanglvl=extended0x not defined).")
    endif()
else()
    message(FATAL_ERROR "Don't know how to check C++11 compatibility with compiler '${CMAKE_CXX_COMPILER_ID}'")
endif()
message(STATUS "Your compiler has C++11 support. Hurray!")

# Enable all compiler warnings
add_to_string_list("${CMAKE_CXX_FLAGS}" CMAKE_CXX_FLAGS "-Wall -Wextra -Wpedantic")

set(3rdparty_INCLUDES)
set(3rdparty_DEPENDEND_LIBS)
set(pfasst_INCLUDES)
set(pfasst_DEPENDEND_LIBS)

if(pfasst_BUILD_TESTS)
    enable_testing()
endif(pfasst_BUILD_TESTS)

# Add / include 3rd-party libraries
message(STATUS "********************************************************************************")
message(STATUS "Configuring 3rd party libraries")
# makes available:
#  - Boost headers in 3rdparty_INCLUDES
#  - Boost libraries in Boost_LIBRARY_DIRS
#  - Eigen headers in 3rdparty_INCLUDES
#  - Google test and mock headers in 3rdparty_INCLUDES (if pfasst_BUILD_TESTS)
#  - FFTW_INCLUDE_PATH (if pfasst_BUILD_EXAMPLES)
#  - FFTW_LIBRARIES (if pfasst_BUILD_EXAMPLES)
add_subdirectory(3rdparty)

message(STATUS "********************************************************************************")
message(STATUS "Configuring sources")
add_subdirectory(include)

if(pfasst_BUILD_EXAMPLES)
    message(STATUS "********************************************************************************")
    message(STATUS "Configuring examples")
    add_subdirectory(examples)
endif()

if(pfasst_BUILD_TESTS)
    message(STATUS "********************************************************************************")
    message(STATUS "Configuring tests")
    add_subdirectory(tests)
endif()
message(STATUS "********************************************************************************")

message(STATUS "C++ Compiler: ${CMAKE_CXX_COMPILER}")
message(STATUS "C++ Compiler ID: ${CMAKE_CXX_COMPILER_ID}")
message(STATUS "C++ Compiler Names: ${CMAKE_CXX_COMPILER_NAMES}")
message(STATUS "C++ Compiler Version: ${CMAKE_CXX_COMPILER_VERSION}")
message(STATUS "C++ Flags: ${CMAKE_CXX_FLAGS}")
message(STATUS "C++ link flags: ${CMAKE_CXX_LINK_FLAGS}")
