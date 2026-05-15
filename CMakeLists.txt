cmake_minimum_required(VERSION 3.10)
project(banking_tests)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TESTS "Build tests" OFF)
option(COLLECT_COVERAGE "Collect coverage" OFF)

add_subdirectory(banking)

if (BUILD_TESTS)
	enable_testing()
	add_subdirectory(third-party/gtest)
	
	file(GLOB TEST_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/tests/*.cpp")
	add_executable(check ${TEST_SOURCES})
	target_link_libraries(check account transaction gtest_main gmock_main)

	if (COLLECT_COVERAGE)
		target_link_libraries(check gcov)
		target_compile_options(check PRIVATE -O0 -g --coverage)
		target_link_options(check PRIVATE --coverage)
	endif()

	add_test(NAME check COMMAND check)
endif()
