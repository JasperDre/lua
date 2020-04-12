cmake_minimum_required(VERSION 3.1)
project(lua C) 

file(GLOB LUA_LIB_SRCS
    "${CMAKE_CURRENT_SOURCE_DIR}/*.c"
)

set(USE_WINDOWS CACHE BOOL ON)
set(USE_POSIX CACHE BOOL OFF)
set(LUA_LIB_INCLUDE ${CMAKE_CURRENT_SOURCE_DIR})

add_library(lua_static STATIC ${LUA_LIB_SRCS})

target_include_directories(lua_static PUBLIC ${LUA_LIB_INCLUDE})

if(USE_POSIX)
  add_definitions(-DLUA_USE_POSIX)
endif()

if(USE_WINDOWS)
  add_definitions(-DLUA_USE_WINDOWS)
endif()
