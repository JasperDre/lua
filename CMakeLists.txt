project(lua C)
cmake_minimum_required(VERSION 3.7)
include(cmake/dist.cmake)
include(lua)

## CONFIGURATION
# Default configuration(we assume POSIX by default)
set(LUA_PATH "LUA_PATH" CACHE STRING "Environment variable to use as package.path.")
set(LUA_CPATH "LUA_CPATH" CACHE STRING "Environment variable to use as package.cpath.")
set(LUA_INIT "LUA_INIT" CACHE STRING "Environment variable for initial script.")

option(LUA_ANSI "Use only ansi features." OFF)
option(LUA_USE_RELATIVE_LOADLIB "Use modified loadlib.c with support for relative paths on posix systems." 
  ON)
set(LUA_IDSIZE "60" CACHE STRING "gives the maximum size for the description of the source.")
set(LUA_PROMPT "> " CACHE STRING "Is the default prompt used by stand-alone Lua.")
set(LUA_PROMPT2 ">> " CACHE STRING "Is the default continuation prompt used by stand-alone Lua.")
set(LUA_MAXINPUT "512" CACHE STRING "Is the maximum length for an input line in the stand-alone interpreter.")

#2DO: LUAI_* and LUAL_* settings, for now defaults are used.
set(LUA_DIRSEP "/")
set(LUA_MODULE_SUFFIX ${CMAKE_SHARED_MODULE_SUFFIX})
set(LUA_LDIR ${INSTALL_LMOD})
set(LUA_CDIR ${INSTALL_CMOD})

if(LUA_USE_RELATIVE_LOADLIB)
  # This will set up relative paths to lib
  string(REGEX REPLACE "[^!/]+" ".." LUA_DIR "!/${INSTALL_BIN}/")
else()
  # Direct path to installation
  set(LUA_DIR ${CMAKE_INSTALL_PREFIX} CACHE STRING "Destination from which modules will be resolved. See INSTALL_LMOD and INSTALL_CMOD.")
endif()

set(LUA_PATH_DEFAULT "./?.lua;${LUA_DIR}${LUA_LDIR}/?.lua;${LUA_DIR}${LUA_LDIR}/?/init.lua;./?/init.lua")
set(LUA_CPATH_DEFAULT "./?${LUA_MODULE_SUFFIX};${LUA_DIR}${LUA_CDIR}/?${LUA_MODULE_SUFFIX};${LUA_DIR}${LUA_CDIR}/loadall${LUA_MODULE_SUFFIX}")

if(WIN32 AND NOT CYGWIN)
  # Windows systems
  option(LUA_WIN "Windows specific build." ON)
  option(LUA_BUILD_WLUA "Build wLua interpretter without console output." ON)
  option(LUA_BUILD_AS_DLL "Build Lua library as Dll." ${BUILD_SHARED_LIBS})

  # Paths(Double escapes needed)
  set(LUA_DIRSEP "\\\\") 
  string(REPLACE " /" ${LUA_DIRSEP} LUA_DIR "${LUA_DIR}")
  string(REPLACE "/" ${LUA_DIRSEP} LUA_LDIR "${LUA_LDIR}")
  string(REPLACE "/" ${LUA_DIRSEP} LUA_CDIR "${LUA_CDIR}")
  string(REPLACE "/" ${LUA_DIRSEP} LUA_PATH_DEFAULT "${LUA_PATH_DEFAULT}")
  string(REPLACE "/" ${LUA_DIRSEP} LUA_CPATH_DEFAULT "${LUA_CPATH_DEFAULT}")
else()
  # Posix systems(incl. Cygwin)
  option(LUA_USE_POSIX "Use POSIX functionality." ON)
  option(LUA_USE_DLOPEN "Use dynamic linker to load modules." ON)
  option(LUA_USE_MKSTEMP "Use mkstep." ON)
  option(LUA_USE_ISATTY "Use tty." ON)
  option(LUA_USE_POPEN "Use popen." ON)
  option(LUA_USE_ULONGJMP "Use ulongjmp" ON)
endif()

## SETUP
# Optional libraries
find_package(Readline)
if(READLINE_FOUND)
  option(LUA_USE_READLINE "Use readline in the Lua CLI." ON)
endif()

find_package(Curses)
if(CURSES_FOUND)
  option(LUA_USE_CURSES "Use curses in the Lua CLI." ON)
endif()

# Setup needed variables and libraries
if(LUA_USE_POSIX)
  # On POSIX Lua links to standard math library "m"
	find_library(MATH_LIBRARY NAMES m)
	if(MATH_LIBRARY)
		list(APPEND LIBS ${MATH_LIBRARY})
	endif()
endif()

if(LUA_USE_DLOPEN)
  # Link to dynamic linker library "dl"
	find_library(DL_LIBRARY NAMES dl)
	if(DL_LIBRARY)
		list(APPEND LIBS ${DL_LIBRARY})
	endif()
endif()

if(LUA_WIN)
  # Add extra rc files to the windows build
  if(MSVC OR MINGW)
    set(LUA_DEF custom/lua.def)
    set(LUA_DLL_RC custom/lua_dll.rc)
    set(LUA_RC custom/lua.rc)
    set(LUAC_RC custom/luac.rc)
  endif()
endif()

if(LUA_USE_READLINE)
  # Add readline
  include_directories(${READLINE_INCLUDE_DIR})
  list(APPEND LIBS ${READLINE_LIBRARY})
endif()

if(LUA_USE_CURSES)
  # Add curses
  include_directories(${CURSES_INCLUDE_DIR})
  list(APPEND LIBS ${CURSES_LIBRARY})
endif()

## SOURCES
# Generate luaconf.h
configure_file(custom/luaconf.h.in ${CMAKE_CURRENT_BINARY_DIR}/luaconf.h)

# Sources and headers
include_directories(src ${CMAKE_CURRENT_BINARY_DIR})
set(SRC_LIB lapi.c lcode.c ldebug.c ldo.c ldump.c lfunc.c 
  lgc.c llex.c lmem.c lobject.c lopcodes.c lparser.c lstate.c 
  lstring.c ltable.c ltm.c lundump.c lvm.c lzio.c lauxlib.c 
  lbaselib.c ldblib.c liolib.c lmathlib.c loslib.c ltablib.c 
  lstrlib.c linit.c)
set(SRC_LUA lua.c)
set(SRC_LUAC luac.c print.c)

if(LUA_USE_RELATIVE_LOADLIB)
  # Use modified loadlib
  list(APPEND SRC_LIB loadlib_rel.c)
else()
  list(APPEND SRC_LIB loadlib.c)
endif()

## BUILD
# Create dynamic library
add_library(liblua SHARED ${SRC_LIB} ${LUA_DLL_RC} ${LUA_DEF})
target_link_libraries(liblua ${LIBS})
set_target_properties(liblua PROPERTIES OUTPUT_NAME lua CLEAN_DIRECT_OUTPUT 1)
if(LUA_BUILD_AS_DLL)
  set_target_properties(liblua PROPERTIES COMPILE_DEFINITIONS LUA_BUILD_AS_DLL)
endif()

# Create static library, this is needed to compile luac in the 5.1.x Lua series
add_library(liblua_static STATIC ${SRC_LIB})
target_link_libraries(liblua_static ${LIBS})

add_executable(lua ${SRC_LUA} ${LUA_RC})
target_link_libraries(lua liblua)

# On windows a variant of the lua interpreter without console output needs to be built
if(LUA_BUILD_WLUA)
  add_executable(wlua WIN32 custom/wmain.c ${SRC_LUA} ${LUA_RC})
  target_link_libraries(wlua liblua)
  install_executable(wlua)
endif()

add_executable(luac ${SRC_LUAC} ${LUAC_RC})
target_link_libraries(luac liblua_static)

install_executable(lua luac)
install_library(liblua)
install_data(README.md COPYRIGHT HISTORY)
install_lua_module(strict strict.lua)
install_header(lua.h lualib.h lauxlib.h lua.hpp ${CMAKE_CURRENT_BINARY_DIR}/luaconf.h)
