cmake_minimum_required(VERSION 3.14)
project(tsrex)

set(CMAKE_CXX_STANDARD 17)

# Main executable
add_executable(tsrex tsrex_core/src/main.cpp)

# Test suite
add_executable(test tsrex_core/test/testrunner.cpp tsrex_core/test/test_smoke.cpp)

