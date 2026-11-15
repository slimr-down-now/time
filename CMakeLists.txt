cmake_minimum_required(VERSION 3.31)
if(CMAKE_HOST_APPLE AND NOT CMAKE_OSX_SYSROOT)
    execute_process(COMMAND xcrun --show-sdk-path
        OUTPUT_VARIABLE CMAKE_OSX_SYSROOT OUTPUT_STRIP_TRAILING_WHITESPACE
        COMMAND_ERROR_IS_FATAL ANY)
    set(CMAKE_OSX_SYSROOT "${CMAKE_OSX_SYSROOT}" CACHE PATH "macOS SDK")
endif()
project(slimr_time VERSION 0.1.0 LANGUAGES CXX)
set(CMAKE_CXX_STANDARD 26)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
include(CTest)
file(MAKE_DIRECTORY "${CMAKE_BINARY_DIR}/test-tmp")
option(ENABLE_SANITIZERS "Instrument tests with ASan and fatal UBSan" ON)

include(CheckCXXCompilerFlag)
check_cxx_compiler_flag("-ftrivial-auto-var-init=zero" SLIMR_HAS_AUTO_INIT)
if(NOT SLIMR_HAS_AUTO_INIT)
    message(FATAL_ERROR "The compiler must support -ftrivial-auto-var-init=zero")
endif()
add_library(slimr_options INTERFACE)
target_compile_options(slimr_options INTERFACE -ftrivial-auto-var-init=zero)

include(CheckCXXSourceCompiles)
check_cxx_source_compiles("
#include <version>
#ifndef _LIBCPP_VERSION
#error not libc++
#endif
int main() {}" SLIMR_USES_LIBCXX)
if(SLIMR_USES_LIBCXX)
    set(CMAKE_REQUIRED_DEFINITIONS -D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_FAST)
    check_cxx_source_compiles("
#include <vector>
#if !defined(_LIBCPP_HARDENING_MODE_FAST) || _LIBCPP_HARDENING_MODE != _LIBCPP_HARDENING_MODE_FAST
#error FAST hardening unavailable
#endif
int main() { std::vector<int> v{1}; return v[0] - 1; }" SLIMR_HAS_FAST_HARDENING)
    unset(CMAKE_REQUIRED_DEFINITIONS)
    if(NOT SLIMR_HAS_FAST_HARDENING)
        message(FATAL_ERROR "libc++ FAST hardening is required")
    endif()
    target_compile_definitions(slimr_options INTERFACE
        _LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_FAST)
endif()
if(ENABLE_SANITIZERS)
    target_compile_options(slimr_options INTERFACE
        -fsanitize=address,undefined -fno-sanitize-recover=all -fno-omit-frame-pointer)
    target_link_options(slimr_options INTERFACE
        -fsanitize=address,undefined -fno-sanitize-recover=all)
endif()
if(CMAKE_BUILD_TYPE STREQUAL "Release")
    include(CheckIPOSupported)
    check_ipo_supported(RESULT SLIMR_HAS_IPO OUTPUT SLIMR_IPO_ERROR LANGUAGES CXX)
    if(NOT SLIMR_HAS_IPO)
        message(FATAL_ERROR "Release IPO unavailable: ${SLIMR_IPO_ERROR}")
    endif()
    set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
    target_compile_options(slimr_options INTERFACE -O3)
endif()

add_library(slimr_time INTERFACE)
add_library(slimr::time ALIAS slimr_time)
target_compile_features(slimr_time INTERFACE cxx_std_26)
target_include_directories(slimr_time INTERFACE "${CMAKE_CURRENT_SOURCE_DIR}")

if(BUILD_TESTING)
    find_package(slimr-test CONFIG REQUIRED)
    add_executable(time_test test.cpp)
    target_link_libraries(time_test PRIVATE slimr::time slimr::test slimr_options)
    add_test(NAME time COMMAND time_test)
    set_tests_properties(time PROPERTIES TIMEOUT 120 ENVIRONMENT "TMPDIR=${CMAKE_BINARY_DIR}/test-tmp")
endif()
