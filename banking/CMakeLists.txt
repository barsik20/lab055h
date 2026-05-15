cmake_minimum_required(VERSION 3.10)
project(banking)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if (NOT TARGET account)
	add_library(account STATIC ${CMAKE_CURRENT_SOURCE_DIR}/Account.cpp)
endif()

if (NOT TARGET transaction)
	add_library(transaction STATIC ${CMAKE_CURRENT_SOURCE_DIR}/Transaction.cpp)
endif()

target_include_directories(account PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_include_directories(transaction PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_link_directories(transaction PUBLIC account)

if (COLLECT_COVERAGE)
	target_compile_options(account PRIVATE -O0 -g --coverage)
	target_link_options(account PRIVATE --coverage)
	target_compile_options(transaction PRIVATE -O0 -g --coverage)
	target_link_options(transaction PRIVATE --coverage)
endif()
