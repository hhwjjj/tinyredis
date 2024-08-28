cmake_minimum_required(VERSION 3.0)

project(tinyRedis)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -std=c++11")

aux_source_directory(. SRC_LIST)

add_executable(test ${SRC_LIST})