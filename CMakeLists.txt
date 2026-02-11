cmake_minimum_required(VERSION 3.10)

set(EXECUTABLE_NAME sslc)

project(${EXECUTABLE_NAME})

option(BUILD_SHARED_LIB "Build shared library (libparser.so) instead of executable" OFF)

# Set 32-bit flag for all targets because it segfaults on 64-bit
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -m32")

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

set(SOURCES
  compile.c
  parse.c
  parselib.c
  extra.c
  gencode.c
  lex.c
  parseext.c
  mcpp_main.c
  mcpp_directive.c
  mcpp_eval.c
  mcpp_expand.c
  mcpp_support.c
  mcpp_system.c
  optimize.c
  compat.c
)

if(BUILD_SHARED_LIB)
  add_library(parser SHARED ${SOURCES})
  target_compile_definitions(parser PRIVATE BUILDING_DLL)

  # Generate version script from exports.def (single source of truth for both Windows and Linux)
  if(NOT ${CMAKE_SYSTEM_NAME} MATCHES "Emscripten")
    file(STRINGS ${CMAKE_SOURCE_DIR}/exports.def _symbols)
    list(FILTER _symbols EXCLUDE REGEX "^(LIBRARY|EXPORTS|$)")
    list(TRANSFORM _symbols STRIP)
    string(REPLACE ";" ";\n    " _symbols_str "${_symbols}")
    file(WRITE ${CMAKE_BINARY_DIR}/exports.map
      "{\n  global:\n    ${_symbols_str};\n  local:\n    *;\n};\n")
    target_link_options(parser PRIVATE "-Wl,--version-script=${CMAKE_BINARY_DIR}/exports.map")
  endif()
else()
  add_executable(${EXECUTABLE_NAME} ${SOURCES})
endif()


if(${CMAKE_SYSTEM_NAME} MATCHES "Emscripten")
  set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g")
  set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g")

  set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O2")
  set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O2")

  set(CMAKE_EXECUTABLE_SUFFIX ".mjs")

  # No need to use asyncify
  # target_link_options(${EXECUTABLE_NAME} PRIVATE "-sASYNCIFY")
  # target_link_options(${EXECUTABLE_NAME} PRIVATE "-sASYNCIFY_STACK_SIZE=20480")

  target_link_options(${EXECUTABLE_NAME} PRIVATE "-sALLOW_MEMORY_GROWTH")
  target_link_options(${EXECUTABLE_NAME} PRIVATE "-MEMORY_GROWTH_LINEAR_STEP=16mb")
  target_link_options(${EXECUTABLE_NAME} PRIVATE "-sNO_DISABLE_EXCEPTION_CATCHING")

  target_link_options( ${EXECUTABLE_NAME} PRIVATE "-lnodefs.js" )
  target_link_options(${EXECUTABLE_NAME} PRIVATE "-sNODERAWFS")

  # Disabled because of emscipten bug
  # target_link_options( ${EXECUTABLE_NAME} PRIVATE "-sCASE_INSENSITIVE_FS" )

  target_link_options(${EXECUTABLE_NAME} PRIVATE "-sMODULARIZE=1")

  target_link_options(${EXECUTABLE_NAME} PRIVATE "-sEXPORT_ES6=1")

  target_link_options(${EXECUTABLE_NAME} PRIVATE "-sENVIRONMENT=node")


  target_link_options(${EXECUTABLE_NAME} PRIVATE "-sEXPORTED_RUNTIME_METHODS=callMain")
  target_link_options(${EXECUTABLE_NAME} PRIVATE "-sEXPORT_ALL")

  target_link_options(${EXECUTABLE_NAME} PRIVATE "-sEXIT_RUNTIME")
  target_link_options(${EXECUTABLE_NAME} PRIVATE "$<$<CONFIG:DEBUG>:-g>")

  target_link_options(${EXECUTABLE_NAME} PRIVATE "-sSTACK_SIZE=2048000")

  set(PACKAGE_JSON_CONTENT [[
  {
    "name": "sslc-emscripten-noderawfs",
    "version": "1.0.0",
    "description": "",
    "main": "sslc.mjs",
    "type": "module",
    "scripts": {
    },
    "bin": "compiler.mjs",
    "author": "",
    "license": "UNLICENSED"
  }

  ]])
  file(WRITE ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/package.json "${PACKAGE_JSON_CONTENT}")

  file(COPY "compiler.mjs" DESTINATION "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}")
endif()

