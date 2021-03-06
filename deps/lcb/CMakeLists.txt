# libcouchbase
# Copyright (C) 2013 Couchbase, Inc
# All rights reserved.
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

CMAKE_MINIMUM_REQUIRED(VERSION 2.8)
# Currently this file is only used for Windows

### BEGIN CONFIGURABLES ###
# These variables can be modified as needed

# Gtest source archive to use
SET(GTEST_ARCHIVE gtest-1.7.0.zip)
# Where to download archive (if it doesn't exist)
SET(GTEST_DLSERVER http://googletest.googlecode.com/files)

# Couchbase mock path to download
SET(COUCHBASE_MOCK_VERSION
    0.8-SNAPSHOT/CouchbaseMock-0.8-20140621.030439-1.jar)

# Maven repository where ${COUCHBASE_MOCK_VERSION} is to be found
SET(COUCHBASE_MOCK_DLSERVER
    http://files.couchbase.com/maven2/org/couchbase/mock/CouchbaseMock)
SET(HAVE_COUCHBASEMOCK 1)
project(libcouchbase)

OPTION(LCB_NO_TESTS "Disable building of tests" OFF)
OPTION(LCB_NO_TOOLS "Disable building of additional tools" OFF)
OPTION(LCB_NO_PLUGINS "Disable the building of IO plugins for external libs" OFF)
OPTION(LCB_MAINTAINER_MODE "Enables maintainer mode" OFF)
OPTION(LCB_NO_SSL "Do not compile SSL support" OFF)
OPTION(LCB_NO_SNAPPY "Do not compile snappy support" ON)
OPTION(LCB_STATIC_SNAPPY "Use the bundled libsnappy. Do not link with system snappy"
    ${WIN32})

### END CONFIGURABLES ###
IF(MSVC)
    SET(CMAKE_DEBUG_POSTFIX "_d")
ENDIF()

SET(LCB_GENSRCDIR ${CMAKE_BINARY_DIR}/generated)
INCLUDE_DIRECTORIES(${LCB_GENSRCDIR})

INCLUDE(cmake/Modules/GetLibcouchbaseFlags.cmake)
INCLUDE(cmake/Modules/FindCouchbaseLibevent.cmake)
INCLUDE(cmake/Modules/FindCouchbaseLibev.cmake)
INCLUDE(cmake/Modules/FindCouchbaseLibuv.cmake)
INCLUDE(cmake/Modules/GetPlatformCCInfo.cmake)
INCLUDE(cmake/Modules/GetVersionInfo.cmake)
INCLUDE(cmake/Modules/GenerateConfigDotH.cmake)
INCLUDE(cmake/Modules/CopyPDB.cmake)
INCLUDE(cmake/Modules/DownloadLcbDep.cmake)
INCLUDE(CheckIncludeFiles)
INCLUDE(cmake/source_files.cmake)


SET(CPACK_PACKAGE_VERSION_MAJOR "${LCB_VERSION_MAJOR}")
SET(CPACK_PACKAGE_VERSION_MINOR "${LCB_VERSION_MINOR}")
SET(CPACK_PACKAGE_VERSION_PATCH "${LCB_VERSION_PATCH}")
SET(CPACK_SOURCE_GENERATOR "TBZ2")
SET(CPACK_SOURCE_PACKAGE_FILE_NAME "libcouchbase-${LCB_VERSION}")
SET(CPACK_SOURCE_IGNORE_FILES "${CMAKE_BINARY_DIR}/;build*;.git/;~$;${CPACK_SOURCE_IGNORE_FILES}")
IF(NOT CMAKE_BUILD_TYPE)
    SET(CPACK_INSTALL_SCRIPT
        "${CMAKE_SOURCE_DIR}/cmake/Modules/DistScript.cmake")
ENDIF()
SET(lcb_package_name "libcouchbase-${LCB_VERSION}_${LCB_ARCH_STRING}_${LCB_CC_STRING}")
SET(CPACK_PACKAGE_FILE_NAME ${lcb_package_name})

IF(WIN32)
    SET(CPACK_GENERATOR "ZIP")
ELSE()
    SET(CPACK_GENERATOR "TGZ")
ENDIF()

INCLUDE(CPack)

SET(CHANGESET ${LCB_VERSION_CHANGESET})
CONFIGURE_FILE(
    ${CMAKE_SOURCE_DIR}/include/libcouchbase/configuration.h.in
    ${LCB_GENSRCDIR}/libcouchbase/configuration.h
    @ONLY)

UNSET(CHANGESET)
CONFIGURE_FILE(
    ${CMAKE_SOURCE_DIR}/win32/dllversion.rc.in
    ${CMAKE_BINARY_DIR}/dllversion.rc)

SET(NT_REQUIRED_VERSION 0x0600)

ADD_DEFINITIONS(-DLIBCOUCHBASE_INTERNAL=1)
ADD_DEFINITIONS(-DCBSASL_STATIC=1)

SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
IF(WIN32)
    SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)
ELSE()
    SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
ENDIF()

SET(SOURCE_ROOT ${CMAKE_SOURCE_DIR})

IF(WIN32)
    ADD_DEFINITIONS(-DWINVER=${NT_REQUIRED_VERSION})
    ADD_DEFINITIONS(-D_WIN32_WINNT=${NT_REQUIRED_VERSION})
    SET(lcb_plat_includes "${SOURCE_ROOT}/win32")
    SET(lcb_plat_libs couchbase_iocp ws2_32.lib)
ELSE()
    ADD_DEFINITIONS(-DLCB_LIBDIR="${CMAKE_LIBRARY_OUTPUT_DIRECTORY}")
    SET(lcb_plat_libs dl m pthread)
    SET(lcb_plat_includes "${SOURCE_ROOT}/cmake")
ENDIF()

INCLUDE_DIRECTORIES(BEFORE ${SOURCE_ROOT}/include
                           ${SOURCE_ROOT}/contrib
                           ${SOURCE_ROOT}/contrib/cbsasl/include
                           ${SOURCE_ROOT}/src
                           ${lcb_plat_includes}
                           ${SOURCE_ROOT})

# These files are bundled in a static library.
# They export no symbols of their own and may
# be considered as 'static'

ADD_LIBRARY(couchbase_utils OBJECT ${LCB_UTILS_SRC})

MACRO(LCB_UTIL tgt)
    SET_TARGET_PROPERTIES(${tgt}
        PROPERTIES
        COMPILE_FLAGS "${LCB_CORE_CFLAGS}"
        POSITION_INDEPENDENT_CODE TRUE)
ENDMACRO()

ADD_LIBRARY(netbuf OBJECT ${LCB_NETBUF_SRC})
ADD_LIBRARY(netbuf-malloc OBJECT ${LCB_NETBUF_SRC})
ADD_LIBRARY(mcreq OBJECT ${LCB_MC_SRC})
ADD_LIBRARY(rdb OBJECT ${LCB_RDB_SRC})
ADD_LIBRARY(lcbio OBJECT ${LCB_IO_SRC})
ADD_LIBRARY(lcbht OBJECT ${LCB_HT_SRC})

SET_TARGET_PROPERTIES(netbuf-malloc PROPERTIES COMPILE_DEFINITIONS NETBUF_LIBC_PROXY=1)
LCB_UTIL(netbuf-malloc)
LCB_UTIL(netbuf)
LCB_UTIL(rdb)
LCB_UTIL(lcbio)
LCB_UTIL(couchbase_utils)
LCB_UTIL(mcreq)
LCB_UTIL(lcbht)

ADD_SUBDIRECTORY(src/vbucket)
ADD_SUBDIRECTORY(contrib/cbsasl)
ADD_SUBDIRECTORY(src/ssl)

IF(LCB_NO_SSL)
    MESSAGE(STATUS "SSL support will be disabled")
    ADD_DEFINITIONS(-DLCB_NO_SSL=1)
ENDIF()

IF(LCB_BUILD_STATIC)
    SET(_lcb_linkspec STATIC)
ELSE()
    SET(_lcb_linkspec SHARED)
ENDIF()


SET(LCB_CORE_OBJS
    $<TARGET_OBJECTS:couchbase_select>
    $<TARGET_OBJECTS:couchbase_utils>
    $<TARGET_OBJECTS:vbucket>
    $<TARGET_OBJECTS:mcreq>
    $<TARGET_OBJECTS:netbuf>
    $<TARGET_OBJECTS:cbsasl>
    $<TARGET_OBJECTS:lcbio>
    $<TARGET_OBJECTS:rdb>
    $<TARGET_OBJECTS:lcbht>
    ${lcb_ssl_objs})

ADD_LIBRARY(couchbaseS STATIC ${LCB_CORE_SRC} ${LCB_CORE_OBJS})
ADD_LIBRARY(couchbase ${_lcb_linkspec} ${LCB_CORE_OBJS} ${LCB_CORE_SRC}
    ${CMAKE_BINARY_DIR}/dllversion.rc)

SET_TARGET_PROPERTIES(couchbase
    PROPERTIES
        COMPILE_FLAGS "${LCB_CORE_CFLAGS}"
        PREFIX "lib"
        IMPORT_PREFIX "lib"
        VERSION "2.0.19"
        SOVERSION "2")

SET_TARGET_PROPERTIES(couchbaseS
    PROPERTIES
       COMPILE_FLAGS "${LCB_CORE_CFLAGS}")

IF(NOT LCB_NO_SNAPPY)
    IF(LCB_STATIC_SNAPPY)
        MESSAGE(STATUS "Will use embedded libsnappy")
        ADD_SUBDIRECTORY(contrib/snappy)
        SET(LCB_SNAPPY_LINK lcbsnappy)
    ELSE()
        INCLUDE(cmake/Modules/FindCouchbaseSnappy.cmake)
        IF(NOT SNAPPY_FOUND)
            MESSAGE(FATAL_ERROR "Snappy not found and building with dynamic snappy requested")
        ELSE()
            SET(LCB_SNAPPY_LINK ${SNAPPY_LIBRARIES})
            INCLUDE_DIRECTORIES(${SNAPPY_INCLUDES})
        ENDIF()
    ENDIF()
ELSE()
    MESSAGE(STATUS "Compression support will be disabled")
    ADD_DEFINITIONS(-DLCB_NO_SNAPPY=1)
    SET(LCB_SNAPPY_LINK "")
ENDIF()

SET_TARGET_PROPERTIES(couchbase PROPERTIES PREFIX "lib")
SET_TARGET_PROPERTIES(couchbase PROPERTIES IMPORT_PREFIX "lib")
SET(LCB_LINK_DEPS
    ${lcb_plat_libs} ${lcb_ssl_libs} ${LCB_SNAPPY_LINK})

TARGET_LINK_LIBRARIES(couchbase ${LCB_LINK_DEPS})
TARGET_LINK_LIBRARIES(couchbaseS ${LCB_LINK_DEPS})

IF(NOT LCB_NO_TESTS)
    ENABLE_TESTING()
    ADD_SUBDIRECTORY(tests tests)
ENDIF(NOT LCB_NO_TESTS)

IF(NOT LCB_NO_TOOLS)
    ADD_SUBDIRECTORY(tools tools)
ENDIF(NOT LCB_NO_TOOLS)

IF(NOT LCB_NO_PLUGINS)
    ADD_SUBDIRECTORY(plugins/io/libuv)
ENDIF()

IF(NOT LCB_NO_PLUGINS AND NOT WIN32)
    ADD_SUBDIRECTORY(plugins/io/libevent)
    ADD_SUBDIRECTORY(plugins/io/libev)
    IF(NOT (HAVE_LIBEV OR HAVE_LIBEVENT))
        MESSAGE(FATAL_ERROR "Must have libevent or libev")
    ENDIF()
ENDIF()


IF(LCB_MAINTAINER_MODE)
    IF(NOT (HAVE_LIBEV AND HAVE_LIBEVENT AND HAVE_LIBUV))
        MESSAGE(FATAL_ERROR "Maintainer mode requires all plugins to be present")
    ENDIF()
ENDIF()

ADD_SUBDIRECTORY(plugins/io/select)
ADD_SUBDIRECTORY(plugins/io/iocp)

INSTALL(TARGETS couchbase
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)

INSTALL(DIRECTORY include/libcouchbase ${LCB_GENSRCDIR}/libcouchbase
    DESTINATION include
    FILES_MATCHING PATTERN *.h PATTERN *.c)

INSTALL_PDBS(couchbase)

ADD_CUSTOM_TARGET(uninstall
    COMMAND xargs rm -vf < ${CMAKE_CURRENT_BINARY_DIR}/install_manifest.txt)
