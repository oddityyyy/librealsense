# License: Apache 2.0. See LICENSE file in root directory.
# Copyright(c) 2019 Intel Corporation. All Rights Reserved.
cmake_minimum_required(VERSION 3.1.3)

project(fw)

file(READ "firmware-version.h" ver)

message(STATUS "Fetching recommended firmwares:")

set(REALSENSE_FIRMWARE_URL "https://librealsense.intel.com" CACHE STRING
    "URL to download firmware binaries from")

string(REGEX MATCH "D4XX_RECOMMENDED_FIRMWARE_VERSION \"([0-9]+.[0-9]+.[0-9]+.[0-9]+)\"" _ ${ver})
set(D4XX_FW_VERSION ${CMAKE_MATCH_1})
#message(STATUS "D4XX_FW_VERSION: ${D4XX_FW_VERSION}")
set(D4XX_FW_SHA1 b4dfa2ceb46e9b79c2d64480b149f188f7538e65)
set(D4XX_FW_URL "${REALSENSE_FIRMWARE_URL}/Releases/RS4xx/FW")

string(REGEX MATCH "SR3XX_RECOMMENDED_FIRMWARE_VERSION \"([0-9]+.[0-9]+.[0-9]+.[0-9]+)\"" _ ${ver})
set(SR3XX_FW_VERSION ${CMAKE_MATCH_1})
#message(STATUS "SR3XX_FW_VERSION: ${SR3XX_FW_VERSION}")
set(SR3XX_FW_SHA1 55237dba5d7db20e7c218975375d05b4210e9460)
set(SR3XX_FW_URL "${REALSENSE_FIRMWARE_URL}/Releases/SR300/FW")

string(REGEX MATCH "L51X_RECOMMENDED_FIRMWARE_VERSION \"([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+)\"" _ ${ver})
set(L51X_FW_VERSION ${CMAKE_MATCH_1})
#message(STATUS "L51X_FW_VERSION: ${L51X_FW_VERSION}")
set(L51X_FW_SHA1 ab73e5bfc520c0aa0340cada4b3e317b8fd31a4d)
set(L51X_FW_URL "${REALSENSE_FIRMWARE_URL}/Releases/L5xx/FW")

add_library(${PROJECT_NAME} STATIC empty.c)

# disable link time optimization for fw by adding -fno-lto to disable -flto flag
# jammy debian has build errors without it
if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  message(STATUS "disable link time optimization for fw project")
  target_compile_options(${PROJECT_NAME} PRIVATE -fno-lto)
endif()

if (MSVC)
  # lib.exe can't handle multiple .res files, so include them in one.
  # even then, the linker won't grab a .res out of a .lib object, so it needs to be explicitly listed
  # and to find the name of the .res file (across cmake generators) we need to create our own rule. :(
  add_custom_command(TARGET ${PROJECT_NAME} BYPRODUCTS ${PROJECT_NAME}.res COMMAND ${CMAKE_RC_COMPILER} ${CMAKE_RC_FLAGS} /I . /fo "${PROJECT_NAME}.res" "${CMAKE_CURRENT_SOURCE_DIR}/${PROJECT_NAME}.rc")
  target_link_libraries(${PROJECT_NAME} PUBLIC "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.res")
endif()

target_include_directories(${PROJECT_NAME} PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
set_target_properties (${PROJECT_NAME} PROPERTIES FOLDER Resources)

function(target_binary url version sha1 symbol ext)
  set(binary "${CMAKE_CURRENT_BINARY_DIR}/${symbol}-${version}${ext}")
  message(STATUS "... ${url}/${symbol}-${version}${ext}")
  file(DOWNLOAD "${url}/${symbol}-${version}${ext}" "${binary}"
       EXPECTED_HASH SHA1=${sha1}
       STATUS status)
  list(GET status 0 error_code)
  if (NOT ${error_code} EQUAL 0)
      message(FATAL_ERROR "FAILED with status ${status}")
  else()
      #message(STATUS "Download firmware ${status} for ${symbol}-${version}${ext}")
  endif()
  string(TOUPPER ${symbol} SYMBOL)
  string(REPLACE "." "," version_commas ${version})
  string(REPLACE "\\" "\\\\" binary_escaped "${binary}")
  configure_file(fw.c.in  "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.c" @ONLY)
  configure_file(fw.h.in  "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.h" @ONLY)
  configure_file(fw.rc.in "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.rc" @ONLY)
  if (MSVC)
    enable_language(RC)
    set_source_files_properties("${PROJECT_NAME}.rc" OBJECT_DEPENDS "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.rc")
    set_source_files_properties("${CMAKE_CURRENT_BINARY_DIR}/${symbol}.rc" OBJECT_DEPENDS "${binary}")
  else()
    set_source_files_properties("${CMAKE_CURRENT_BINARY_DIR}/${symbol}.c" OBJECT_DEPENDS "${binary}")
  endif()
  target_sources(${PROJECT_NAME} PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/${symbol}.c")
endfunction()

target_binary( "${D4XX_FW_URL}"  "${D4XX_FW_VERSION}"  "${D4XX_FW_SHA1}" D4XX_FW_Image .bin)
target_binary( "${SR3XX_FW_URL}"  "${SR3XX_FW_VERSION}"  "${SR3XX_FW_SHA1}" SR3XX_FW_Image .bin)
target_binary( "${L51X_FW_URL}"  "${L51X_FW_VERSION}"  "${L51X_FW_SHA1}" L51X_FW_Image .bin)


install(TARGETS ${PROJECT_NAME} EXPORT realsense2Targets
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

