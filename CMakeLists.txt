cmake_minimum_required( VERSION 2.6 )
PROJECT( Assimp )

# Define here the needed parameters
set (ASSIMP_VERSION_MAJOR 2)
set (ASSIMP_VERSION_MINOR 0)
set (ASSIMP_VERSION_PATCH 0)
set (ASSIMP_VERSION ${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}.${ASSIMP_VERSION_PATCH})
set (ASSIMP_SOVERSION ${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR})
SET ( PROJECT_VERSION "${ASSIMP_SOVERSION}" )

set(PACKAGE_VERSION "0" CACHE STRING "the package-specific version used for uploading the sources")

option(OPT_BUILD_PACKAGES "Set to ON to generate CPack configuration files and packaging targets" OFF)
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules")
set(CPACK_COMPONENTS_ALL assimp-bin libassimp${ASSIMP_VERSION_MAJOR} assimp-dev)

if(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
  add_definitions(-fPIC) # this is a very important switch and some libraries seem now to have it....
  ## hide all not-exported symbols
  add_definitions( -fvisibility=hidden )
endif()

INCLUDE (FindPkgConfig)
INCLUDE_DIRECTORIES( include )

# If this is an in-source build (CMAKE_SOURCE_DIR == CMAKE_BINARY_DIR),
# write the library/executable files to the respective directories in the
# source tree. During an out-of-source build, however, do not litter this
# directory, since that is probably what the user wanted to avoid.
IF ( CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR )
	SET( CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_HOME_DIRECTORY}/lib )
	SET( CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_HOME_DIRECTORY}/bin )
ENDIF ( CMAKE_SOURCE_DIR STREQUAL CMAKE_BINARY_DIR )

# Cache these to allow the user to override them manually.
SET( LIB_INSTALL_DIR "lib" CACHE PATH
	"Path the built library files are installed to." )
SET( INCLUDE_INSTALL_DIR "include" CACHE PATH
	"Path the header files are installed to." )
SET( BIN_INSTALL_DIR "bin" CACHE PATH
	"Path the tool executables are installed to." )

# Generate a pkg-config .pc for the Assimp library.
CONFIGURE_FILE( "${PROJECT_SOURCE_DIR}/assimp.pc.in" "${PROJECT_BINARY_DIR}/assimp.pc" @ONLY )
INSTALL( FILES "${PROJECT_BINARY_DIR}/assimp.pc" DESTINATION ${LIB_INSTALL_DIR}/pkgconfig/ COMPONENT assimp-dev)

# Globally enbale Boost resp. the Boost workaround – it is also needed by the
# tools which include the Assimp headers.
SET ( ENABLE_BOOST_WORKAROUND OFF CACHE BOOL
	"If a simple implementation of the used Boost functions is used. Slightly reduces functionality, but enables builds without Boost available."
)
IF ( ENABLE_BOOST_WORKAROUND )
	INCLUDE_DIRECTORIES( code/BoostWorkaround )
	ADD_DEFINITIONS( -DASSIMP_BUILD_BOOST_WORKAROUND )
	MESSAGE( STATUS "Building a non-boost version of Assimp." )
ELSE ( ENABLE_BOOST_WORKAROUND )
	SET( Boost_DETAILED_FAILURE_MSG ON )
	FIND_PACKAGE( Boost 1.35 )

	IF ( NOT Boost_FOUND )
		MESSAGE( FATAL_ERROR
			"Boost libraries (http://www.boost.org/) not found. "
			"You can build a non-boost version of Assimp with slightly reduced "
			"functionality by specifying -DENABLE_BOOST_WORKAROUND=ON."
		)
	ENDIF ( NOT Boost_FOUND )

	INCLUDE_DIRECTORIES( ${Boost_INCLUDE_DIRS} )
ENDIF ( ENABLE_BOOST_WORKAROUND )


SET ( NO_EXPORT OFF CACHE BOOL
	"Disable Assimp's export functionality." 
)

# Search for external dependencies, and build them from source if not found
# Search for zlib
find_package(ZLIB)
if( NOT ZLIB_FOUND )
  message(STATUS "compiling zlib from souces")
  include(CheckIncludeFile)
  include(CheckTypeSize)
  include(CheckFunctionExists)
  # compile from sources
  add_subdirectory(contrib/zlib)
  set(ZLIB_FOUND 1)
  set(ZLIB_LIBRARIES zlib)
  set(ZLIB_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/3rdparty/zlib)
else(NOT ZLIB_FOUND)
  ADD_DEFINITIONS(-DASSIMP_BUILD_NO_OWN_ZLIB)
endif(NOT ZLIB_FOUND)
INCLUDE_DIRECTORIES(${ZLIB_INCLUDE_DIR})

# Search for unzip
if (PKG_CONFIG_FOUND)
	PKG_CHECK_MODULES(UNZIP minizip)
endif (PKG_CONFIG_FOUND)

IF ( NO_EXPORT )
	ADD_DEFINITIONS( -DASSIMP_BUILD_NO_EXPORT)
	MESSAGE( STATUS "Build an import-only version of Assimp." )
ENDIF( NO_EXPORT )


SET ( ASSIMP_BUILD_ARCHITECTURE "" CACHE STRING 
	"describe the current architecture."
)
IF    ( ASSIMP_BUILD_ARCHITECTURE STREQUAL "")
ELSE  ( ASSIMP_BUILD_ARCHITECTURE STREQUAL "")
	ADD_DEFINITIONS ( -D'ASSIMP_BUILD_ARCHITECTURE="${ASSIMP_BUILD_ARCHITECTURE}"' )
ENDIF ( ASSIMP_BUILD_ARCHITECTURE STREQUAL "")

SET ( ASSIMP_BUILD_COMPILER "" CACHE STRING 
	"describe the current compiler."
)
IF    ( ASSIMP_BUILD_COMPILER STREQUAL "")
ELSE  ( ASSIMP_BUILD_COMPILER STREQUAL "")
	ADD_DEFINITIONS ( -D'ASSIMP_BUILD_COMPILER="${ASSIMP_BUILD_COMPILER}"' )
ENDIF ( ASSIMP_BUILD_COMPILER STREQUAL "")

MARK_AS_ADVANCED ( ASSIMP_BUILD_ARCHITECTURE ASSIMP_BUILD_COMPILER )


ADD_SUBDIRECTORY( code/ )
SET ( BUILD_ASSIMP_TOOLS ON CACHE BOOL
	"If the supplementary tools for Assimp are built in addition to the library."
)
IF ( BUILD_ASSIMP_TOOLS )
	IF ( WIN32 )
		ADD_SUBDIRECTORY( tools/assimp_view/ )
		ADD_SUBDIRECTORY( samples/SimpleOpenGL/ )
		ADD_SUBDIRECTORY( samples/SimpleTexturedOpenGL/ )
	ENDIF ( WIN32 )
	ADD_SUBDIRECTORY( tools/assimp_cmd/ )
ENDIF ( BUILD_ASSIMP_TOOLS )

SET ( BUILD_TESTS OFF CACHE BOOL
	"If the test suite for Assimp is built in addition to the library."
)
IF ( BUILD_TESTS )
	IF ( WIN32 )
		ADD_SUBDIRECTORY( test/ )
	ELSE ( WIN32 )
		MESSAGE( WARNING "The Assimp test suite is currently Windows-only." )
	ENDIF ( WIN32 )
ENDIF ( BUILD_TESTS )

if(CMAKE_CPACK_COMMAND AND UNIX AND OPT_BUILD_PACKAGES)
  # Packing information
  set(CPACK_PACKAGE_NAME assimp{ASSIMP_VERSION_MAJOR})
  set(CPACK_PACKAGE_CONTACT "" CACHE STRING "Package maintainer and PGP signer.")
  set(CPACK_PACKAGE_VENDOR "http://assimp.sourceforge.net/")
  set(CPACK_PACKAGE_DISPLAY_NAME "Assimp ${ASSIMP_VERSION}.${ASSIMP_VERSION_MINOR}")
  set(CPACK_PACKAGE_DESCRIPTION_SUMMARY " - Open Asset Import Library")
  set(CPACK_PACKAGE_VERSION ${ASSIMP_VERSION}.${PACKAGE_VERSION})
  set(CPACK_PACKAGE_VERSION_MAJOR ${ASSIMP_VERSION_MAJOR})
  set(CPACK_PACKAGE_VERSION_MINOR ${ASSIMP_VERSION_MINOR})
  set(CPACK_PACKAGE_VERSION_PATCH ${ASSIMP_VERSION_PATCH})
  set(CPACK_PACKAGE_INSTALL_DIRECTORY "assimp${ASSIMP_VERSION_MAJOR}.${ASSIMP_VERSION_MINOR}")
  #set(CPACK_PACKAGE_DESCRIPTION_FILE ${CMAKE_CURRENT_SOURCE_DIR}/description)
  set(CPACK_RESOURCE_FILE_LICENSE ${CMAKE_CURRENT_SOURCE_DIR}/LICENSE)

  set(CPACK_COMPONENT_ASSIMP-BIN_DISPLAY_NAME "tools")
  set(CPACK_COMPONENT_ASSIMP-DEV_DISPLAY_NAME "common headers and installs")
  set(CPACK_COMPONENT_LIBASSIMP${ASSIMP_VERSION_MAJOR}_DISPLAY_NAME "libraries")
  set(CPACK_COMPONENT_ASSIMP-BIN_DEPENDS libassimp${ASSIMP_VERSION_MAJOR})
  set(CPACK_COMPONENT_ASSIMP-DEV_DEPENDS libassimp${ASSIMP_VERSION_MAJOR})
  set(CPACK_DEBIAN_BUILD_DEPENDS debhelper cmake libboost-dev libboost-thread-dev libboost-math-dev zlib1g-dev pkg-config)

  # debian
  set(CPACK_DEBIAN_PACKAGE_PRIORITY optional)
  set(CPACK_DEBIAN_PACKAGE_SECTION libs)
  set(CPACK_DEBIAN_PACKAGE_DEPENDS ${CPACK_COMPONENTS_ALL})
  set(CPACK_DEBIAN_PACKAGE_SUGGESTS)
  set(CPACK_DEBIAN_PACKAGE_NAME assimp)
  set(CPACK_DEBIAN_PACKAGE_REMOVE_SOURCE_FILES contrib/cppunit-1.12.1 contrib/cppunit_note.txt contrib/zlib workspaces test doc obj )
  set(CPACK_DEBIAN_PACKAGE_SOURCE_COPY svn export --force)
  set(CPACK_DEBIAN_CHANGELOG)
  execute_process(COMMAND lsb_release -is
    OUTPUT_VARIABLE _lsb_distribution OUTPUT_STRIP_TRAILING_WHITESPACE
    RESULT_VARIABLE _lsb_release_failed)
  set(CPACK_DEBIAN_DISTRIBUTION_NAME ${_lsb_distribution} CACHE STRING "Name of the distrubiton")
  string(TOLOWER ${CPACK_DEBIAN_DISTRIBUTION_NAME} CPACK_DEBIAN_DISTRIBUTION_NAME)
  if( ${CPACK_DEBIAN_DISTRIBUTION_NAME} STREQUAL "ubuntu" )
    set(CPACK_DEBIAN_DISTRIBUTION_RELEASES karmic lucid maverick natty CACHE STRING "Release code-names of the distrubiton release")
  endif()
  set(DPUT_HOST "" CACHE STRING "PPA repository to upload the debian sources")
  include(CPack)
  include(DebSourcePPA)
endif()
