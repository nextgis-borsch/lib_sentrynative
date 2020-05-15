################################################################################
# Copyright (C) 2020, NextGIS <info@nextgis.com>
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################

cmake_minimum_required(VERSION 2.8.12 FATAL_ERROR)
project(sentrynative)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK AND (BUILD_SHARED_LIBS OR BUILD_STATIC_LIBS))
  message(FATAL_ERROR "Only OSX_FRAMEWORK key or any or both BUILD_SHARED_LIBS
                       and BUILD_STATIC_LIBS keys are permitted")
endif()

if(OSX_FRAMEWORK)
    set(INSTALL_BIN_DIR "bin" CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR "Library/Frameworks" CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_CMAKECONF_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Resources/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
    set(INSTALL_INC_DIR ${INSTALL_LIB_DIR}/${PROJECT_NAME}.framework/Headers CACHE INTERNAL "Installation directory for headers" FORCE)
    set(SKIP_INSTALL_HEADERS ON)
    set(SKIP_INSTALL_EXECUTABLES ON)
    set(SKIP_INSTALL_FILES ON)
    set(SKIP_INSTALL_EXPORT ON)
    set(CMAKE_MACOSX_RPATH ON)
else()
    include(GNUInstallDirs)

    set(INSTALL_BIN_DIR ${CMAKE_INSTALL_BINDIR} CACHE INTERNAL "Installation directory for executables" FORCE)
    set(INSTALL_LIB_DIR ${CMAKE_INSTALL_LIBDIR} CACHE INTERNAL "Installation directory for libraries" FORCE)
    set(INSTALL_INC_DIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE INTERNAL "Installation directory for headers" FORCE)
    set(INSTALL_PKGCONFIG_DIR "${INSTALL_LIB_DIR}/pkgconfig" CACHE INTERNAL "Installation directory for pkgconfig (.pc) files" FORCE)
    set(INSTALL_CMAKECONF_DIR ${CMAKE_INSTALL_DATADIR}/${PROJECT_NAME}/CMake CACHE INTERNAL "Installation directory for cmake config files" FORCE)
endif()

set(PACKAGE_NAME ${PROJECT_NAME})
string(TOUPPER ${PACKAGE_NAME} PACKAGE_UPPER_NAME)

set(LIB_NAME ${PROJECT_NAME})
set(TARGETS ${LIB_NAME})

include(util)

check_version(MAJOR_VER MINOR_VER REL_VER FIX_VER)
set(VERSION ${MAJOR_VER}.${MINOR_VER}.${REL_VER})
report_version(${PROJECT_NAME} ${VERSION})

if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
	set(LINUX TRUE)
endif()

option(SENTRY_PIC "Build sentry (and dependent) libraries as position independent libraries" ON)

# CMAKE_POSITION_INDEPENDENT_CODE must be set BEFORE adding any libraries (including subprojects)
if(SENTRY_PIC)
	set(CMAKE_POSITION_INDEPENDENT_CODE ON)
else()
	set(CMAKE_POSITION_INDEPENDENT_CODE OFF)
endif()

if(WIN32)
	set(SENTRY_DEFAULT_TRANSPORT "winhttp")
elseif(APPLE OR LINUX)
	set(SENTRY_DEFAULT_TRANSPORT "curl")
else()
	set(SENTRY_DEFAULT_TRANSPORT "none")
endif()

set(SENTRY_TRANSPORT ${SENTRY_DEFAULT_TRANSPORT} CACHE STRING
  "The HTTP transport that sentry uses to submit events to the sentry server, can be either 'none', 'curl' or 'winhttp' on windows.")

if(SENTRY_TRANSPORT STREQUAL "winhttp")
	set(SENTRY_TRANSPORT_WINHTTP TRUE)
elseif(SENTRY_TRANSPORT STREQUAL "curl")
	set(SENTRY_TRANSPORT_CURL TRUE)
elseif(SENTRY_TRANSPORT STREQUAL "none")
	set(SENTRY_TRANSPORT_NONE TRUE)
else()
	message(FATAL_ERROR "SENTRY_TRANSPORT must be one of 'none', 'curl' or 'winhttp'")
endif()

if(SENTRY_TRANSPORT_WINHTTP AND NOT WIN32)
	message(FATAL_ERROR "The winhttp transport is only supported on Windows.")
endif()

set(SENTRY_BACKEND_NONE TRUE)

message(STATUS "SENTRY_TRANSPORT=${SENTRY_TRANSPORT}")

option(WITH_ASAN_OPTION "Build sentry-native with address sanitizer" OFF)
if(WITH_ASAN_OPTION)
	add_compile_options(-g -fsanitize=address -fno-omit-frame-pointer)
	link_libraries(-fsanitize=address)
endif()

option(WITH_TSAN_OPTION "Build sentry-native with thread sanitizer" OFF)
if(WITH_TSAN_OPTION)
	add_compile_options(-g -fsanitize=thread -fno-omit-frame-pointer)
	link_libraries(-fsanitize=thread)
endif()

# use -O3 when doing `RelWithDebInfo` builds
if(NOT MSVC)
	foreach(lang ASM C CXX)
		string(REPLACE "-O2" "-O3" CMAKE_${lang}_FLAGS_RELWITHDEBINFO "${CMAKE_${lang}_FLAGS_RELWITHDEBINFO}")
	endforeach()
endif()

# helper function to add sources to existing TARGET prepended with ${CMAKE_CURRENT_SOURCE_DIR}/${SUBDIR}
function(sentry_target_sources_cwd TARGET)
	cmake_parse_arguments(STSC "" "SUBDIR" "" ${ARGN})
	foreach(src ${STSC_UNPARSED_ARGUMENTS})
		if(IS_ABSOLUTE "${src}")
			target_sources(${TARGET} PRIVATE ${src})
		else()
			target_sources(${TARGET} PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/${STSC_SUBDIR}/${src}")
		endif()
	endforeach()
endfunction()

include(FindAnyProject)

# ===== sentry library =====

add_library(${LIB_NAME} "${PROJECT_SOURCE_DIR}/vendor/mpack.c")
add_library(sentry::sentry ALIAS ${LIB_NAME})
add_subdirectory(src)

if(BUILD_SHARED_LIBS)
	target_compile_definitions(${LIB_NAME} PRIVATE SENTRY_BUILD_SHARED)
else()
	target_compile_definitions(${LIB_NAME} PUBLIC SENTRY_BUILD_STATIC)
endif()

if(SENTRY_TRANSPORT_CURL)
	find_anyproject(CURL REQUIRED)
	target_include_directories(${LIB_NAME} PRIVATE ${CURL_INCLUDE_DIR})
	# The exported sentry target must not contain any path of the build machine, therefore use generator expressions
	# FIXME: cmake 3.12 introduced the target CURL::libcurl
	string(REPLACE ";" "$<SEMICOLON>" GENEX_CURL_LIBRARIES "${CURL_LIBRARIES}")
	string(REPLACE ";" "$<SEMICOLON>" GENEX_CURL_COMPILE_DEFINITIONS "${CURL_COMPILE_DEFINITIONS}")
	target_link_libraries(${LIB_NAME} PRIVATE $<BUILD_INTERFACE:${GENEX_CURL_LIBRARIES}>)
	target_compile_definitions(${LIB_NAME} PRIVATE $<BUILD_INTERFACE:${GENEX_CURL_COMPILE_DEFINITIONS}>)
elseif(SENTRY_TRANSPORT_WINHTTP)
	target_link_libraries(${LIB_NAME} PRIVATE winhttp)
endif()

set_property(TARGET ${LIB_NAME} PROPERTY C_VISIBILITY_PRESET hidden)
if(MSVC)
	if(CMAKE_SIZEOF_VOID_P EQUAL 4)
		set(CMAKE_ASM_MASM_FLAGS "${CMAKE_ASM_MASM_FLAGS} /safeseh")
	endif()
else()
	target_compile_options(${LIB_NAME} PRIVATE $<BUILD_INTERFACE:-Wall -Wextra -Wpedantic>)
	# ignore all warnings for mpack
	set_source_files_properties(
		"${PROJECT_SOURCE_DIR}/vendor/mpack.c"
		PROPERTIES
		COMPILE_FLAGS
		"-w"
	)
endif()

target_include_directories(${LIB_NAME}
	PUBLIC
		"$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>"
	PRIVATE
		"$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/src>"
)

#respect CMAKE_SYSTEM_VERSION
if(WIN32)
	if(${CMAKE_SYSTEM_VERSION} MATCHES "^10")
		target_compile_definitions(${LIB_NAME} PRIVATE "_WIN32_WINNT=0x0A00")
	elseif(${CMAKE_SYSTEM_VERSION} MATCHES "^6.3")
		target_compile_definitions(${LIB_NAME} PRIVATE "_WIN32_WINNT=0x0603")
	elseif(${CMAKE_SYSTEM_VERSION} MATCHES "^6.2")
		target_compile_definitions(${LIB_NAME} PRIVATE "_WIN32_WINNT=0x0602")
	elseif(${CMAKE_SYSTEM_VERSION} MATCHES "^6.1")
		target_compile_definitions(${LIB_NAME} PRIVATE "_WIN32_WINNT=0x0601")
	elseif(${CMAKE_SYSTEM_VERSION} MATCHES "^6.0")
		target_compile_definitions(${LIB_NAME} PRIVATE "_WIN32_WINNT=0x0600")
	elseif(${CMAKE_SYSTEM_VERSION} MATCHES "^5.2")
		target_compile_definitions(${LIB_NAME} PRIVATE "_WIN32_WINNT=0x0502")
	elseif(${CMAKE_SYSTEM_VERSION} MATCHES "^5.1")
		target_compile_definitions(${LIB_NAME} PRIVATE "_WIN32_WINNT=0x0501")
	endif()
	target_link_libraries(${LIB_NAME} PRIVATE shlwapi)
endif()

if(LINUX)
	target_link_libraries(${LIB_NAME} PRIVATE pthread dl)
elseif(ANDROID)
	target_link_libraries(${LIB_NAME} PRIVATE dl log)
elseif(MSVC)
	target_link_libraries(${LIB_NAME} PRIVATE dbghelp)
elseif(MINGW)
	target_link_libraries(${LIB_NAME} PRIVATE dbghelp)
	target_compile_options(${LIB_NAME} PRIVATE
		-Wno-unused-variable
		-Wno-unused-parameter
		-Wno-format
		-Wno-incompatible-pointer-types
		-Wno-incompatible-function-pointer-types
	)
endif()

add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)

# Export package ===============================================================

# Add all targets to the build-tree export set
export(TARGETS ${TARGETS}
   FILE ${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Targets.cmake)

if(REGISTER_PACKAGE)
   # Export the package for use from the build-tree
   # (this registers the build-tree with a global CMake-registry)
   export(PACKAGE ${PACKAGE_UPPER_NAME})
endif()

# Create the SENTRYConfig.cmake file
configure_file(cmake/PackageConfig.cmake.in
	${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake @ONLY)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
	install(TARGETS ${TARGETS} 
		EXPORT ${PACKAGE_UPPER_NAME}Targets
		RUNTIME DESTINATION ${INSTALL_BIN_DIR}
		ARCHIVE DESTINATION ${INSTALL_LIB_DIR}
		LIBRARY DESTINATION ${INSTALL_LIB_DIR}
		INCLUDES DESTINATION ${INSTALL_INC_DIR}
		FRAMEWORK DESTINATION ${INSTALL_LIB_DIR}
	)
	
	install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/include/sentry.h"
		DESTINATION "${INSTALL_INC_DIR}"
	)

	# Install the <Package>Config.cmake
	install(FILES
		${PROJECT_BINARY_DIR}/${PACKAGE_UPPER_NAME}Config.cmake
		DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)

	# Install the export set for use with the install-tree
	install(EXPORT ${PACKAGE_UPPER_NAME}Targets DESTINATION ${INSTALL_CMAKECONF_DIR} COMPONENT dev)
endif()

# Archiving ====================================================================

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
get_cpack_filename(${VERSION} PROJECT_CPACK_FILENAME)
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)
