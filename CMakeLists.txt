# Open Asset Import Library (assimp)
# ----------------------------------------------------------------------
# 
# Copyright (c) 2006-2016, assimp team
# All rights reserved.
#
# Redistribution and use of this software in source and binary forms,
# with or without modification, are permitted provided that the
# following conditions are met:
#
# * Redistributions of source code must retain the above
#   copyright notice, this list of conditions and the
#   following disclaimer.
#
# * Redistributions in binary form must reproduce the above
#   copyright notice, this list of conditions and the
#   following disclaimer in the documentation and/or other
#   materials provided with the distribution.
#
# * Neither the name of the assimp team, nor the names of its
#   contributors may be used to endorse or promote products
#   derived from this software without specific prior
#   written permission of the assimp team.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
# A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
# OWNER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
# SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
# LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
# DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
# THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
#
#----------------------------------------------------------------------
SET(CMAKE_LEGACY_CYGWIN_WIN32 0) # Remove when CMake >= 2.8.4 is required
cmake_minimum_required( VERSION 3.0 )

include("cmake/HunterGate.cmake")
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.14.26.tar.gz"
    SHA1 "45ac03944db2bc7fc55dfceca42d89ec7fa1dca2"
)

PROJECT( Assimp VERSION 3.2.0)

OPTION(BUILD_SHARED_LIBS "Build package with shared libraries." ON)
IF(NOT BUILD_SHARED_LIBS)
  SET(LINK_SEARCH_START_STATIC TRUE)
ENDIF(NOT BUILD_SHARED_LIBS)

# Define here the needed parameters
SET (ASSIMP_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
SET (ASSIMP_VERSION_MINOR ${PROJECT_VERSION_MINOR})
SET (ASSIMP_VERSION_PATCH ${PROJECT_VERSION_PATCH}) # subversion revision?
SET (ASSIMP_VERSION ${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}.${ASSIMP_VERSION_PATCH})
SET (ASSIMP_SOVERSION 3)
SET (PROJECT_VERSION "${ASSIMP_VERSION}")

SET(ASSIMP_PACKAGE_VERSION "0" CACHE STRING "the package-specific version used for uploading the sources")

if(NOT HUNTER_ENABLED)
  # Needed for openddl_parser config, no use of c++11 at this moment
  add_definitions( -DOPENDDL_NO_USE_CPP11 )
  set_property( GLOBAL PROPERTY CXX_STANDARD 11 )
endif()

# Get the current working branch
EXECUTE_PROCESS(
  COMMAND git rev-parse --abbrev-ref HEAD
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  OUTPUT_VARIABLE GIT_BRANCH
  OUTPUT_STRIP_TRAILING_WHITESPACE
  ERROR_QUIET
)

# Get the latest abbreviated commit hash of the working branch
EXECUTE_PROCESS(
  COMMAND git log -1 --format=%h
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  OUTPUT_VARIABLE GIT_COMMIT_HASH
  OUTPUT_STRIP_TRAILING_WHITESPACE
  ERROR_QUIET
)

IF(NOT GIT_COMMIT_HASH)
  SET(GIT_COMMIT_HASH 0)
ENDIF(NOT GIT_COMMIT_HASH)

configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/revision.h.in
  ${CMAKE_CURRENT_BINARY_DIR}/revision.h
)

include_directories(${CMAKE_CURRENT_BINARY_DIR})

OPTION(ASSIMP_OPT_BUILD_PACKAGES "Set to ON to generate CPack configuration files and packaging targets" OFF)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules")
SET(LIBASSIMP_COMPONENT     "libassimp${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}.${ASSIMP_VERSION_PATCH}" )
SET(LIBASSIMP-DEV_COMPONENT "libassimp${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}.${ASSIMP_VERSION_PATCH}-dev" )
SET(CPACK_COMPONENTS_ALL assimp-bin ${LIBASSIMP_COMPONENT} ${LIBASSIMP-DEV_COMPONENT} assimp-dev)
SET(ASSIMP_LIBRARY_SUFFIX "" CACHE STRING "Suffix to append to library names")

OPTION(ASSIMP_ANDROID_JNIIOSYSTEM "Android JNI IOSystem support is active" OFF)

# Workaround to be able to deal with compiler bug "Too many sections" with mingw.
IF( CMAKE_COMPILER_IS_MINGW )
  ADD_DEFINITIONS(-DASSIMP_BUILD_NO_IFC_IMPORTER )
ENDIF()

IF((CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX) AND NOT CMAKE_COMPILER_IS_MINGW)
  IF (BUILD_SHARED_LIBS AND CMAKE_SIZEOF_VOID_P EQUAL 8) # -fPIC is only required for shared libs on 64 bit
     SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
  ENDIF()
  # hide all not-exported symbols
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden -Wall -std=c++0x" )
ELSEIF(MSVC)
  # enable multi-core compilation with MSVC
  add_compile_options(/MP)
ELSEIF ( "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" )
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden -Wall -Wno-long-long -pedantic -std=c++11" )
ELSEIF( CMAKE_COMPILER_IS_MINGW )
	SET( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden -Wall -Wno-long-long -pedantic -std=c++11" )
ENDIF()

INCLUDE (FindPkgConfig)
INCLUDE_DIRECTORIES( include )

INCLUDE (PrecompiledHeader)

# If this is an in-source build (CMAKE_SOURCE_DIR == CMAKE_BINARY_DIR),
# write the library/executable files to the respective directories in the
# source tree. During an out-of-source build, however, do not litter this
# directory, since that is probably what the user wanted to avoid.
IF ( CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR )
  SET( CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_HOME_DIRECTORY}/lib" )
  SET( CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_HOME_DIRECTORY}/lib" )
  SET( CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_HOME_DIRECTORY}/bin" )
ENDIF ( CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR )

# Cache these to allow the user to override them manually.
SET( ASSIMP_LIB_INSTALL_DIR "lib" CACHE PATH
  "Path the built library files are installed to." )
SET( ASSIMP_INCLUDE_INSTALL_DIR "include" CACHE PATH
  "Path the header files are installed to." )
SET( ASSIMP_BIN_INSTALL_DIR "bin" CACHE PATH
  "Path the tool executables are installed to." )

IF (CMAKE_BUILD_TYPE STREQUAL "Debug")
  SET(CMAKE_DEBUG_POSTFIX "d" CACHE STRING "Debug Postfix for lib, samples and tools")
ELSE()
  SET(CMAKE_DEBUG_POSTFIX "" CACHE STRING "Debug Postfix for lib, samples and tools")
ENDIF()

# Only generate this target if no higher-level project already has
IF (NOT TARGET uninstall)
  # add make uninstall capability
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules/cmake_uninstall.cmake.in" "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake" IMMEDIATE @ONLY)
  add_custom_target(uninstall "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")
ENDIF()

FIND_PACKAGE( DirectX )

OPTION ( ASSIMP_NO_EXPORT
  "Disable Assimp's export functionality."
  OFF
)

IF( CMAKE_COMPILER_IS_GNUCXX )
  SET(LIBSTDC++_LIBRARIES -lstdc++)
ENDIF( CMAKE_COMPILER_IS_GNUCXX )

hunter_add_package(ZLIB)
find_package(ZLIB CONFIG REQUIRED)

SET(ZLIB_LIBRARIES_LINKED -lz)

# Search for unzip
hunter_add_package(minizip)
find_package(minizip CONFIG REQUIRED)
set(UNZIP_FOUND TRUE)

IF ( ASSIMP_NO_EXPORT )
  ADD_DEFINITIONS( -DASSIMP_BUILD_NO_EXPORT)
  MESSAGE( STATUS "Build an import-only version of Assimp." )
ENDIF( ASSIMP_NO_EXPORT )

SET ( ASSIMP_BUILD_ARCHITECTURE "" CACHE STRING
  "describe the current architecture."
)
IF    ( ASSIMP_BUILD_ARCHITECTURE STREQUAL "")
ELSE  ( ASSIMP_BUILD_ARCHITECTURE STREQUAL "")
  ADD_DEFINITIONS ( -D'ASSIMP_BUILD_ARCHITECTURE="${ASSIMP_BUILD_ARCHITECTURE}"' )
ENDIF ( ASSIMP_BUILD_ARCHITECTURE STREQUAL "")

# ${CMAKE_GENERATOR}
SET ( ASSIMP_BUILD_COMPILER "" CACHE STRING
  "describe the current compiler."
)
IF    ( ASSIMP_BUILD_COMPILER STREQUAL "")
ELSE  ( ASSIMP_BUILD_COMPILER STREQUAL "")
  ADD_DEFINITIONS ( -D'ASSIMP_BUILD_COMPILER="${ASSIMP_BUILD_COMPILER}"' )
ENDIF ( ASSIMP_BUILD_COMPILER STREQUAL "")

MARK_AS_ADVANCED ( ASSIMP_BUILD_ARCHITECTURE ASSIMP_BUILD_COMPILER )


SET ( ASSIMP_BUILD_NONFREE_C4D_IMPORTER OFF CACHE BOOL
  "Build the C4D importer, which relies on the non-free Melange SDK."
)

IF (ASSIMP_BUILD_NONFREE_C4D_IMPORTER)
  IF ( MSVC )
    SET(C4D_INCLUDES "${CMAKE_CURRENT_SOURCE_DIR}/contrib/Melange/_melange/includes")

    # pick the correct prebuilt library
    IF(MSVC11)
      SET(C4D_LIB_POSTFIX "_2012md")
    ELSEIF(MSVC10)
      SET(C4D_LIB_POSTFIX "_2010md")
    ELSEIF(MSVC90)
      SET(C4D_LIB_POSTFIX "_2008md")
    ELSE()
      MESSAGE( FATAL_ERROR
        "C4D is currently only supported with MSVC 9, 10, 11"
      )
    ENDIF()

    IF(CMAKE_CL_64)
      SET(C4D_LIB_ARCH_POSTFIX "_x64")
    ELSE()
      SET(C4D_LIB_ARCH_POSTFIX "")
    ENDIF()

    SET(C4D_LIB_BASE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/contrib/Melange/_melange/lib/WIN")

    SET(C4D_DEBUG_LIBRARY "${C4D_LIB_BASE_PATH}/debug/_melange_lib${C4D_LIB_ARCH_POSTFIX}${C4D_LIB_POSTFIX}.lib")
    SET(C4D_RELEASE_LIBRARY "${C4D_LIB_BASE_PATH}/release/_melange_lib${C4D_LIB_ARCH_POSTFIX}${C4D_LIB_POSTFIX}.lib")

    # winsock and winmm are necessary dependencies of melange (this is undocumented, but true.)
    SET(C4D_EXTRA_LIBRARIES WSock32.lib Winmm.lib)
  ELSE ()
    MESSAGE( FATAL_ERROR
      "C4D is currently only available on Windows with melange SDK installed in contrib/Melange"
    )
  ENDIF ( MSVC )
ELSE (ASSIMP_BUILD_NONFREE_C4D_IMPORTER)
  ADD_DEFINITIONS( -DASSIMP_BUILD_NO_C4D_IMPORTER )
ENDIF (ASSIMP_BUILD_NONFREE_C4D_IMPORTER)

####
# Installation (https://github.com/forexample/package-example) {

set(config_install_dir "lib/cmake/${PROJECT_NAME}")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

# Configuration
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

# Include module with fuction 'write_basic_package_version_file'
include(CMakePackageConfigHelpers)

# Note: PROJECT_VERSION is used as a VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Use variables:
#   * targets_export_name
#   * PROJECT_NAME
configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

install(
    EXPORT "${targets_export_name}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)

# }

ADD_SUBDIRECTORY( code/ )
option ( ASSIMP_BUILD_ASSIMP_TOOLS
  "If the supplementary tools for Assimp are built in addition to the library."
  ON
)
IF ( ASSIMP_BUILD_ASSIMP_TOOLS )
  IF ( WIN32 )
    option ( ASSIMP_BUILD_ASSIMP_VIEW "If the Assimp view tool is built. (requires DirectX)" ${DirectX_FOUND} )
    IF ( ASSIMP_BUILD_ASSIMP_VIEW )
      ADD_SUBDIRECTORY( tools/assimp_view/ )
    ENDIF ( ASSIMP_BUILD_ASSIMP_VIEW )
  ENDIF ( WIN32 )
  
  ADD_SUBDIRECTORY( tools/assimp_cmd/ )
ENDIF ( ASSIMP_BUILD_ASSIMP_TOOLS )

option ( ASSIMP_BUILD_SAMPLES
  "If the official samples are built as well (needs Glut)."
  OFF
)

IF ( ASSIMP_BUILD_SAMPLES)
  IF ( WIN32 )
    ADD_SUBDIRECTORY( samples/SimpleTexturedOpenGL/ )
  ENDIF ( WIN32 )
  ADD_SUBDIRECTORY( samples/SimpleOpenGL/ )
ENDIF ( ASSIMP_BUILD_SAMPLES )

OPTION ( ASSIMP_BUILD_TESTS
  "If the test suite for Assimp is built in addition to the library."
  ON
)

IF ( ASSIMP_BUILD_TESTS )
  ADD_SUBDIRECTORY( test/ )
ENDIF ( ASSIMP_BUILD_TESTS )

IF(MSVC)
  OPTION ( ASSIMP_INSTALL_PDB
    "Install MSVC debug files."
    ON
  )
ENDIF(MSVC)

# Generate a pkg-config .pc for the Assimp library.
CONFIGURE_FILE( "${PROJECT_SOURCE_DIR}/assimp.pc.in" "${PROJECT_BINARY_DIR}/assimp.pc" @ONLY )
INSTALL( FILES "${PROJECT_BINARY_DIR}/assimp.pc" DESTINATION ${ASSIMP_LIB_INSTALL_DIR}/pkgconfig/ COMPONENT ${LIBASSIMP-DEV_COMPONENT})

IF(CMAKE_CPACK_COMMAND AND UNIX AND ASSIMP_OPT_BUILD_PACKAGES)
  # Packing information
  SET(CPACK_PACKAGE_NAME                    "assimp{ASSIMP_VERSION_MAJOR}")
  SET(CPACK_PACKAGE_CONTACT "" CACHE STRING "Package maintainer and PGP signer.")
  SET(CPACK_PACKAGE_VENDOR                  "https://github.com/assimp")
  SET(CPACK_PACKAGE_DISPLAY_NAME            "Assimp ${ASSIMP_VERSION}")
  SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY     " - Open Asset Import Library ${ASSIMP_VERSION}")
  SET(CPACK_PACKAGE_VERSION                 "${ASSIMP_VERSION}.${ASSIMP_PACKAGE_VERSION}" )
  SET(CPACK_PACKAGE_VERSION_MAJOR           "${ASSIMP_VERSION_MAJOR}")
  SET(CPACK_PACKAGE_VERSION_MINOR           "${ASSIMP_VERSION_MINOR}")
  SET(CPACK_PACKAGE_VERSION_PATCH           "${ASSIMP_VERSION_PATCH}")
  SET(CPACK_PACKAGE_INSTALL_DIRECTORY       "assimp${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}")
  SET(CPACK_RESOURCE_FILE_LICENSE           "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")

  string(TOUPPER ${LIBASSIMP_COMPONENT}     "LIBASSIMP_COMPONENT_UPPER")
  string(TOUPPER ${LIBASSIMP-DEV_COMPONENT} "LIBASSIMP-DEV_COMPONENT_UPPER")

  SET(CPACK_COMPONENT_ASSIMP-BIN_DISPLAY_NAME                       "tools")
  SET(CPACK_COMPONENT_ASSIMP-BIN_DEPENDS                            "${LIBASSIMP_COMPONENT}" )
  SET(CPACK_COMPONENT_${LIBASSIMP_COMPONENT_UPPER}_DISPLAY_NAME     "libraries")
  SET(CPACK_COMPONENT_${LIBASSIMP-DEV_COMPONENT_UPPER}_DISPLAY_NAME "common headers and installs")
  SET(CPACK_COMPONENT_${LIBASSIMP-DEV_COMPONENT_UPPER}_DEPENDS $    "{LIBASSIMP_COMPONENT}" )
  SET(CPACK_COMPONENT_ASSIMP-DEV_DISPLAY_NAME                       "${CPACK_COMPONENT_${LIBASSIMP-DEV_COMPONENT}_DISPLAY_NAME}" )
  SET(CPACK_COMPONENT_ASSIMP-DEV_DEPENDS                            "${LIBASSIMP-DEV_COMPONENT}" )
  SET(CPACK_DEBIAN_BUILD_DEPENDS debhelper cmake zlib1g-dev pkg-config)

  # debian
  SET(CPACK_DEBIAN_PACKAGE_PRIORITY "optional")
  SET(CPACK_DEBIAN_CMAKE_OPTIONS    "-DBUILD_ASSIMP_SAMPLES:BOOL=${ASSIMP_BUILD_SAMPLES}")
  SET(CPACK_DEBIAN_PACKAGE_SECTION  "libs" )
  SET(CPACK_DEBIAN_PACKAGE_DEPENDS  "${CPACK_COMPONENTS_ALL}")
  SET(CPACK_DEBIAN_PACKAGE_SUGGESTS)
  SET(CPACK_DEBIAN_PACKAGE_NAME     "assimp")
  SET(CPACK_DEBIAN_PACKAGE_REMOVE_SOURCE_FILES contrib/cppunit-1.12.1 contrib/cppunit_note.txt contrib/zlib workspaces test doc obj samples packaging)
  SET(CPACK_DEBIAN_PACKAGE_SOURCE_COPY svn export --force)
  SET(CPACK_DEBIAN_CHANGELOG)
  execute_process(COMMAND lsb_release -is
    OUTPUT_VARIABLE _lsb_distribution OUTPUT_STRIP_TRAILING_WHITESPACE
    RESULT_VARIABLE _lsb_release_failed)
  SET(CPACK_DEBIAN_DISTRIBUTION_NAME ${_lsb_distribution} CACHE STRING "Name of the distrubiton")
  STRING(TOLOWER ${CPACK_DEBIAN_DISTRIBUTION_NAME} CPACK_DEBIAN_DISTRIBUTION_NAME)
  IF( ${CPACK_DEBIAN_DISTRIBUTION_NAME} STREQUAL "ubuntu" )
    SET(CPACK_DEBIAN_DISTRIBUTION_RELEASES lucid maverick natty oneiric precise CACHE STRING "Release code-names of the distrubiton release")
  ENDIF()
  SET(DPUT_HOST "" CACHE STRING "PPA repository to upload the debian sources")
  include(CPack)
  include(DebSourcePPA)
ENDIF()

if(WIN32)
	if (CMAKE_SIZEOF_VOID_P EQUAL 8)
		SET(BIN_DIR "${PROJECT_SOURCE_DIR}/bin64/")
		SET(LIB_DIR "${PROJECT_SOURCE_DIR}/lib64/")
	elseif()
		SET(BIN_DIR "${PROJECT_SOURCE_DIR}/bin32/")
		SET(LIB_DIR "${PROJECT_SOURCE_DIR}/lib32/")
	ENDIF()

	if(MSVC12)
		SET(ASSIMP_MSVC_VERSION "vc120")
	elseif(MSVC14)
		SET(ASSIMP_MSVC_VERSION "vc140")
	ENDIF(MSVC12)

	if(MSVC12 OR MSVC14)
		add_custom_target(UpdateAssimpLibsDebugSymbolsAndDLLs COMMENT "Copying Assimp Libraries ..." VERBATIM)
		add_custom_command(TARGET UpdateAssimpLibsDebugSymbolsAndDLLs COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/code/Release/assimp-${ASSIMP_MSVC_VERSION}-mt.dll	${BIN_DIR}assimp-${ASSIMP_MSVC_VERSION}-mt.dll VERBATIM)
		add_custom_command(TARGET UpdateAssimpLibsDebugSymbolsAndDLLs COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/code/Release/assimp-${ASSIMP_MSVC_VERSION}-mt.exp	${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mt.exp VERBATIM)
		add_custom_command(TARGET UpdateAssimpLibsDebugSymbolsAndDLLs COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/code/Release/assimp-${ASSIMP_MSVC_VERSION}-mt.lib	${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mt.lib VERBATIM)
		add_custom_command(TARGET UpdateAssimpLibsDebugSymbolsAndDLLs COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/code/Debug/assimp-${ASSIMP_MSVC_VERSION}-mtd.dll		${BIN_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.dll  VERBATIM)
		add_custom_command(TARGET UpdateAssimpLibsDebugSymbolsAndDLLs COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/code/Debug/assimp-${ASSIMP_MSVC_VERSION}-mtd.exp		${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.exp VERBATIM)
		add_custom_command(TARGET UpdateAssimpLibsDebugSymbolsAndDLLs COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/code/Debug/assimp-${ASSIMP_MSVC_VERSION}-mtd.ilk		${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.ilk VERBATIM)
		add_custom_command(TARGET UpdateAssimpLibsDebugSymbolsAndDLLs COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/code/Debug/assimp-${ASSIMP_MSVC_VERSION}-mtd.lib		${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.lib VERBATIM)
		add_custom_command(TARGET UpdateAssimpLibsDebugSymbolsAndDLLs COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/code/Debug/assimp-${ASSIMP_MSVC_VERSION}-mtd.pdb		${LIB_DIR}assimp-${ASSIMP_MSVC_VERSION}-mtd.pdb VERBATIM)
	ENDIF(MSVC12 OR MSVC14)
ENDIF (WIN32)
