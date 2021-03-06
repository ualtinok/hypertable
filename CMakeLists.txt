#
# Copyright (C) 2009 Doug Judd (Zvents, Inc.)
#
# This program is free software; you can redistribute it and/or
# modify it under the terms of the GNU General Public License
# as published by the Free Software Foundation; either version 2
# of the License, or any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA
# 02110-1301, USA.
#

project(HYPERTABLE)

# Hypertable version components
set(VERSION_MAJOR "0")
set(VERSION_MINOR "9")
set(VERSION_MICRO "2")
set(VERSION_PATCH "4")

# cmake behavior compatibility
cmake_minimum_required(VERSION 2.4.6)
mark_as_advanced(CMAKE_BACKWARDS_COMPATIBILITY)

# saner control structure syntax
set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS true)

if (COMMAND cmake_policy)
  # we prefer the more strict behavior, to find out more:
  # cmake --help-policy CMP0003
  cmake_policy(SET CMP0003 NEW)
endif ()

# Something nice for GUI like ccmake
if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Debug CACHE STRING
      "Options: None Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif ()

# install directory prefix
if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX "/opt/hypertable"
      CACHE PATH "installation directory prefix" FORCE)
endif ()

# Enable testing
enable_testing()

# A simple test helper
set(TEST_DIFF ${HYPERTABLE_SOURCE_DIR}/bin/src-utils/testdiff)

# gcc warning settings
if (CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
  add_definitions(-Wall -pedantic -Wno-long-long)
endif ()

add_definitions(-D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64 -D_REENTRANT)

# Uncomment the following to build dynamic libraries
# set (BUILD_SHARED_LIBS ON)

# Locate required packages
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

# A more informative hypertable version string
include(VersionHelper)

# Install directory
set(INSTALL_DIR ${CMAKE_INSTALL_PREFIX}/${VERSION})

find_package(Threads REQUIRED)
message(STATUS "Use thread library: ${CMAKE_THREAD_LIBS_INIT}")
find_package(Boost REQUIRED)
find_package(BerkeleyDB REQUIRED)
find_package(EXPAT REQUIRED)
find_package(Log4cpp REQUIRED)
find_package(Readline REQUIRED)
find_package(SIGAR REQUIRED)
find_package(ZLIB REQUIRED)
find_package(Doxygen)
find_package(Tcmalloc)
find_package(GoogleHash)
find_package(Kfs)
find_package(Ant)
find_package(JNI)
find_package(PythonLibs)
find_package(LibEvent)
find_package(Thrift)
find_package(RubyThrift)
find_package(PHP5Thrift)
find_package(PerlThrift)
find_package(PythonThrift)
find_package(JavaThrift)

if (PYTHON_LIBS_FOUND)
  message (STATUS "Python libraries found at : ${PYTHON_LIBS_FOUND}")
endif ()

if (JAVA_INCLUDE_PATH)
  message(STATUS "Java headers found at: ${JAVA_INCLUDE_PATH}")
  if (HADOOP_INCLUDE_PATH)
    message(STATUS "Hadoop includes located at: ${HADOOP_INCLUDE_PATH}")
  else ()
    message(STATUS "Please set HADOOP_INCLUDE_PATH variable for Hadoop support")
  endif(HADOOP_INCLUDE_PATH)

  if (HADOOP_LIB_PATH)
    message(STATUS "Hadoop libraries located at: ${HADOOP_LIB_PATH}")
  else ()
    message(STATUS "Please set HADOOP_LIB_PATH variable for Hadoop support")
  endif ()

  if (NOT BUILD_SHARED_LIBS)
    message(STATUS "Not building shared libraries. "
            "Hadoop map-reduce support will be disabled")
  endif ()
else ()
  message(STATUS "Java JNI not found. Hadoop support will be disabled.")
endif(JAVA_INCLUDE_PATH)

# set malloc library except in Debug mode, as tcmalloc makes program startup
# extremely slow, which slows down debug cycles.
if (NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
  set(MALLOC_LIBRARY ${Tcmalloc_LIBRARIES})
endif ()

# use GoogleHash if found
if (GoogleHash_FOUND)
  include_directories(${GoogleHash_INCLUDE_DIR})
  add_definitions(-DHT_WITH_GOOGLE_HASH)
endif (GoogleHash_FOUND)

# include directories
include_directories(src/cc ${HYPERTABLE_BINARY_DIR}/src/cc
    ${ZLIB_INCLUDE_DIR} ${Boost_INCLUDE_DIRS} ${Log4cpp_INCLUDE_DIR}
    ${EXPAT_INCLUDE_DIRS} ${BDB_INCLUDE_DIR} ${READLINE_INCLUDE_DIR}
    ${SIGAR_INCLUDE_DIR})

if (Thrift_FOUND)
  include_directories(${LibEvent_INCLUDE_DIR} ${Thrift_INCLUDE_DIR})
  add_definitions(-DHT_WITH_THRIFT)
  set(ThriftBroker_IDL_DIR ${HYPERTABLE_SOURCE_DIR}/src/cc/ThriftBroker)
endif ()

if (BOOST_VERSION MATCHES "1_34")
  message(STATUS "Got boost 1.34.x, prepend fix directory")
  include_directories(BEFORE src/cc/boost-1_34-fix)
endif ()

if (GCC_VERSION MATCHES "^([4-9]|[1-9][0-9]+)\\.")
  add_definitions(-Wno-variadic-macros)
  # turn off deprecated feature warnings for gcc 4.3+
  if (GCC_VERSION MATCHES "^([4-9]|[1-9][0-9]+)\\.([3-9]|[1-9][0-9]+)")
    add_definitions(-Wno-deprecated)
  endif ()
elseif (GCC_VERSION MATCHES "^([4-9]|[1-9][0-9]+)\\.")
  message(STATUS "Ancient gcc detected, hope it finishes despite warnings...")
endif ()

include(TestHelper)
include(DocHelper)

# sub-project directories
add_subdirectory(src)
add_subdirectory(examples)
add_subdirectory(contrib)
add_subdirectory(tests/integration)

# Java stuff
if (ANT_FOUND)
  include(JavaHelper)
endif ()

install(FILES demo/metadata-test.xml DESTINATION ${VERSION}/demo)
install(FILES demo/testdata.txt.bz2 DESTINATION ${VERSION}/demo)
install(FILES demo/Test.xml DESTINATION ${VERSION}/demo)
install(FILES demo/Test-schema.xml DESTINATION ${VERSION}/demo)
install(FILES conf/hypertable.cfg DESTINATION ${VERSION}/conf)
install(FILES conf/METADATA.xml DESTINATION ${VERSION}/conf)
install(FILES lib/commons-logging-1.0.4.jar DESTINATION ${VERSION}/java/lib)
install(FILES lib/hadoop-0.20.0-core.jar DESTINATION ${VERSION}/java/lib)
install(FILES lib/junit-4.3.1.jar DESTINATION ${VERSION}/java/lib)
install(FILES lib/log4j-1.2.13.jar DESTINATION ${VERSION}/java/lib)
install(FILES lib/hypertable-${VERSION}.jar DESTINATION ${VERSION}/java/lib)
install(FILES ${HYPERTABLE_BINARY_DIR}/java/hypertable-${VERSION}.jar
        DESTINATION ${VERSION}/java/lib OPTIONAL)
install(PROGRAMS bin/jrun bin/clean-database.sh bin/start-dfsbroker.sh
                 bin/start-master.sh bin/start-hyperspace.sh
                 bin/start-rangeserver.sh bin/start-all-servers.sh
                 bin/stop-servers.sh bin/start-thriftbroker.sh
                 bin/random-wait.sh
        DESTINATION ${VERSION}/bin)
install(DIRECTORY examples/apache_log DESTINATION ${VERSION}/examples
        USE_SOURCE_PERMISSIONS)
install(DIRECTORY examples/hql_tutorial DESTINATION ${VERSION}/examples
        USE_SOURCE_PERMISSIONS)
install(DIRECTORY examples/freebase DESTINATION ${VERSION}/examples
        USE_SOURCE_PERMISSIONS)
