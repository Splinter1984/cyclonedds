#
# Copyright(c) 2006 to 2022 ZettaScale Technology and others
#
# This program and the accompanying materials are made available under the
# terms of the Eclipse Public License v. 2.0 which is available at
# http://www.eclipse.org/legal/epl-2.0, or the Eclipse Distribution License
# v. 1.0 which is available at
# http://www.eclipse.org/org/documents/edl-v10.php.
#
# SPDX-License-Identifier: EPL-2.0 OR BSD-3-Clause
#
include(GenerateExportHeader)
include(CheckCSourceCompiles)
include(CheckLibraryExists)
include(CheckIncludeFile)
include(CheckTypeSize)
include(CheckSymbolExists)

# Lightweight IP stack can be used on non-embedded targets too, but the
# runtime must be instructed to use it instead of the native stack. Of course
# for embedded targets there is no "native" stack and the runtime module must
# always be instructed to use an "alternative" stack.
option(WITH_LWIP "Use lightweight IP stack" OFF)
option(WITH_FREERTOS "Build for FreeRTOS" OFF)

set(source_dir "${CMAKE_CURRENT_SOURCE_DIR}")
set(binary_dir "${CMAKE_CURRENT_BINARY_DIR}")

add_library(ddsrt INTERFACE)
target_include_directories(
  ddsrt INTERFACE
    "$<BUILD_INTERFACE:${source_dir}/include>"
    "$<BUILD_INTERFACE:${source_dir}/src>"
    "$<BUILD_INTERFACE:${binary_dir}/include>")

set(headers
  "${binary_dir}/include/dds/config.h"
  "${source_dir}/include/dds/ddsrt/avl.h"
  "${source_dir}/include/dds/ddsrt/bits.h"
  "${source_dir}/include/dds/ddsrt/fibheap.h"
  "${source_dir}/include/dds/ddsrt/hopscotch.h"
  "${source_dir}/include/dds/ddsrt/log.h"
  "${source_dir}/include/dds/ddsrt/retcode.h"
  "${source_dir}/include/dds/ddsrt/attributes.h"
  "${source_dir}/include/dds/ddsrt/endian.h"
  "${source_dir}/include/dds/ddsrt/arch.h"
  "${source_dir}/include/dds/ddsrt/misc.h"
  "${source_dir}/include/dds/ddsrt/mh3.h"
  "${source_dir}/include/dds/ddsrt/io.h"
  "${source_dir}/include/dds/ddsrt/process.h"
  "${source_dir}/include/dds/ddsrt/sched.h"
  "${source_dir}/include/dds/ddsrt/strtod.h"
  "${source_dir}/include/dds/ddsrt/strtol.h"
  "${source_dir}/include/dds/ddsrt/types.h"
  "${source_dir}/include/dds/ddsrt/countargs.h"
  "${source_dir}/include/dds/ddsrt/foreach.h"
  "${source_dir}/include/dds/ddsrt/static_assert.h"
  "${source_dir}/include/dds/ddsrt/circlist.h"
  "${source_dir}/include/dds/ddsrt/atomics.h"
  "${source_dir}/include/dds/ddsrt/atomics/arm.h"
  "${source_dir}/include/dds/ddsrt/atomics/gcc.h"
  "${source_dir}/include/dds/ddsrt/atomics/msvc.h"
  "${source_dir}/include/dds/ddsrt/atomics/sun.h"
  "${source_dir}/include/dds/ddsrt/dynlib.h"
  "${source_dir}/include/dds/ddsrt/environ.h"
  "${source_dir}/include/dds/ddsrt/heap.h"
  "${source_dir}/include/dds/ddsrt/ifaddrs.h"
  "${source_dir}/include/dds/ddsrt/md5.h"
  "${source_dir}/include/dds/ddsrt/netstat.h"
  "${source_dir}/include/dds/ddsrt/string.h"
  "${source_dir}/include/dds/ddsrt/sockets.h"
  "${source_dir}/src/sockets_priv.h"
  "${source_dir}/include/dds/ddsrt/threads.h"
  "${source_dir}/src/threads_priv.h"
  "${source_dir}/include/dds/ddsrt/cdtors.h"
  "${source_dir}/include/dds/ddsrt/random.h"
  "${source_dir}/include/dds/ddsrt/align.h")

set(sources
  "${source_dir}/src/atomics.c"
  "${source_dir}/src/avl.c"
  "${source_dir}/src/dynlib.c"
  "${source_dir}/src/bits.c"
  "${source_dir}/src/bswap.c"
  "${source_dir}/src/io.c"
  "${source_dir}/src/log.c"
  "${source_dir}/src/retcode.c"
  "${source_dir}/src/strtod.c"
  "${source_dir}/src/strtol.c"
  "${source_dir}/src/mh3.c"
  "${source_dir}/src/environ.c"
  "${source_dir}/src/expand_vars.c"
  "${source_dir}/src/fibheap.c"
  "${source_dir}/src/hopscotch.c"
  "${source_dir}/src/circlist.c"
  "${source_dir}/src/threads.c"
  "${source_dir}/src/string.c"
  "${source_dir}/src/sockets.c"
  "${source_dir}/src/md5.c"
  "${source_dir}/src/xmlparser.c"
  "${source_dir}/src/ifaddrs.c"
  "${source_dir}/src/cdtors.c"
  "${source_dir}/src/random.c"
  "${source_dir}/src/time.c")

# Not every target offers the same set of features. For embedded targets the
# set of features may even be different between builds. e.g. a FreeRTOS build
# could use the lightweight IP stack, but later change to FreeRTOS+TCP.
#
# Most features and target specific settings can be determined at compile time
# by a combination of pre-defined macros. However, some features require input
# from the build system. e.g. that the target operating system is FreeRTOS or
# that the network stack to be used is lwIP as opposed to the target native
# network stack. In order to mix-and-match various compilers, architectures,
# operating systems, etc input from the build system is required.
if(WITH_LWIP)
  set(CMAKE_EXTRA_INCLUDE_FILES "ws2tcpip.h")
  set(hostname_header "unistd.h")
  set(inet_header "lwip/sockets.h")
  set(netdb_header "lwip/netdb.h")
  set(sockopt_header "lwip/sockets.h")
  list(APPEND headers
    "${source_dir}/include/dds/ddsrt/sockets/posix.h")
  list(APPEND sources
    "${source_dir}/src/ifaddrs/lwip/ifaddrs.c"
    "${source_dir}/src/sockets/posix/socket.c"
    "${source_dir}/src/sockets/posix/gethostname.c")
else()
  if(WIN32)
    if(POLICY CMP0075)
      # suppress warnings concerning a change in behaviour of check_include_files in
      # combination with CMAKE_REQUIRED_LIBRARIES
      cmake_policy(SET CMP0075 NEW)
    endif()
    set(CMAKE_EXTRA_INCLUDE_FILES "ws2tcpip.h")
    set(CMAKE_REQUIRED_LIBRARIES ws2_32 iphlpapi)
    set(hostname_header "winsock2.h")
    set(inet_header "ws2tcpip.h")
    set(netdb_header "ws2tcpip.h")
    set(sockopt_header "ws2tcpip.h")
    list(APPEND headers
      "${source_dir}/include/dds/ddsrt/sockets/windows.h")
    list(APPEND sources
      "${source_dir}/src/ifaddrs/windows/ifaddrs.c"
      "${source_dir}/src/sockets/windows/socket.c"
      "${source_dir}/src/sockets/windows/gethostname.c")
  else()
    set(CMAKE_EXTRA_INCLUDE_FILES "netinet/in.h")
    set(hostname_header "unistd.h")
    set(inet_header "arpa/inet.h")
    set(netdb_header "netdb.h")
    set(sockopt_header "netinet/in.h")
    list(APPEND headers
      "${source_dir}/include/dds/ddsrt/sockets/posix.h")
    list(APPEND sources
      "${source_dir}/src/sockets/posix/socket.c"
      "${source_dir}/src/sockets/posix/gethostname.c")
    if(WITH_ZEPHYR)
      list(APPEND sources
        "${source_dir}/src/ifaddrs/zephyr/ifaddrs.c")
    else()
      list(APPEND sources
        "${source_dir}/src/ifaddrs/posix/ifaddrs.c")
    endif()
  endif()
endif()

check_symbol_exists("gethostname" ${hostname_header} DDSRT_HAVE_GETHOSTNAME)
check_symbol_exists("inet_ntop" ${inet_header} DDSRT_HAVE_INET_NTOP)
check_symbol_exists("inet_pton" ${inet_header} DDSRT_HAVE_INET_PTON)
check_symbol_exists("getaddrinfo" ${netdb_header} DDSRT_HAVE_GETADDRINFO)
check_symbol_exists("gethostbyname_r" ${netdb_header} DDSRT_HAVE_GETHOSTBYNAME_R)
if(DDSRT_HAVE_GETADDRINFO OR DDSRT_HAVE_GETHOSTBYNAME_R)
  set(DDSRT_HAVE_DNS TRUE)
endif()
set(DDSRT_HAVE_IPV6 FALSE)
if(ENABLE_IPV6)
  check_type_size("struct sockaddr_in6" SIZEOF_SOCKADDR_IN6)
  if(NOT ENABLE_IPV6 STREQUAL "AUTO")
    if(SIZEOF_SOCKADDR_IN6)
      set(DDSRT_HAVE_IPV6 TRUE)
    else()
      message(FATAL_ERROR "ENABLE_IPV6 requires IPv6 to be supported by the platform")
    endif()
  elseif(SIZEOF_SOCKADDR_IN6)
    set(DDSRT_HAVE_IPV6 TRUE)
    set(ENABLE_IPV6 ON)
  else() 
    set(ENABLE_IPV6 OFF)
 endif()
endif()
if(DDSRT_HAVE_IPV6)
  message(STATUS "Building with IPv6 support")
else()
  message(STATUS "Building without IPv6 support")
endif()
set(DDSRT_HAVE_SSM FALSE)
if(ENABLE_SOURCE_SPECIFIC_MULTICAST)
  check_symbol_exists("IP_ADD_SOURCE_MEMBERSHIP" ${sockopt_header} DDSRT_HAVE_IP_ADD_SOURCE_MEMBERSHIP)
  if(DDSRT_HAVE_IPV6)
    check_symbol_exists("MCAST_JOIN_SOURCE_GROUP" ${sockopt_header} DDSRT_HAVE_MCAST_JOIN_SOURCE_GROUP)
  else()
    # Set MCAST_JOIN_SOURCE_GROUP if IPv6 is disabled to simplify the logic here
    set(DDSRT_HAVE_MCAST_JOIN_SOURCE_GROUP TRUE)
  endif()
  if(NOT ENABLE_SOURCE_SPECIFIC_MULTICAST STREQUAL "AUTO")
    if(DDSRT_HAVE_IP_ADD_SOURCE_MEMBERSHIP AND DDSRT_HAVE_MCAST_JOIN_SOURCE_GROUP)
      set(DDSRT_HAVE_SSM TRUE)
    else()
      message(FATAL_ERROR "ENABLE_SOURCE_SPECIFIC_MULTICAST requires SSM to be supported by the platform")
    endif()
  elseif(DDSRT_HAVE_IP_ADD_SOURCE_MEMBERSHIP AND DDSRT_HAVE_MCAST_JOIN_SOURCE_GROUP)
    set(DDSRT_HAVE_SSM TRUE)
    set(ENABLE_SOURCE_SPECIFIC_MULTICAST ON)
  else()
    set(ENABLE_SOURCE_SPECIFIC_MULTICAST OFF)
  endif()
endif()
if(DDSRT_HAVE_SSM)
  message(STATUS "Building with source-specific multicast support")
else()
  message(STATUS "Building without source-specific multicast support")
endif()

if(WITH_FREERTOS)
  list(APPEND headers
    "${source_dir}/include/dds/ddsrt/sync/freertos.h"
    "${source_dir}/include/dds/ddsrt/threads/freertos.h"
    "${source_dir}/include/dds/ddsrt/time/freertos.h")
  list(APPEND sources
    "${source_dir}/src/environ/posix/environ.c"
    "${source_dir}/src/heap/freertos/heap.c"
    "${source_dir}/src/sync/freertos/sync.c"
    "${source_dir}/src/sync/freertos/tasklist.c"
    "${source_dir}/src/threads/freertos/threads.c"
    "${source_dir}/src/time/freertos/time.c"
    "${source_dir}/src/process/freertos/process.c"
    "${source_dir}/src/random/posix/random.c")

  string(CONCAT rusage_code
    "#include <FreeRTOS.h>\n"
    "#if configUSE_TRACE_FACILITY == 1 && configGENERATE_RUN_TIME_STATS == 1\n"
    "int main(int argc, char *argv[]) { return 0; }\n"
    "#endif\n")
  check_c_source_compiles("${rusage_code}" DDSRT_HAVE_RUSAGE)
  if(DDSRT_HAVE_RUSAGE)
    list(APPEND sources
      "${source_dir}/src/rusage/freertos/rusage.c")
  endif()
elseif(WITH_ZEPHYR)
  set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
  find_package(Threads REQUIRED)
  target_link_libraries(ddsrt INTERFACE Threads::Threads)

  list(APPEND headers
    "${source_dir}/include/dds/ddsrt/sync/posix.h"
    "${source_dir}/include/dds/ddsrt/threads/posix.h"
    "${source_dir}/include/dds/ddsrt/types/posix.h")
  list(APPEND sources
    "${source_dir}/src/environ/zephyr/environ.c"
    "${source_dir}/src/heap/posix/heap.c"
    "${source_dir}/src/process/posix/process.c"
    "${source_dir}/src/random/posix/random.c"
    "${source_dir}/src/sync/posix/sync.c"
    "${source_dir}/src/threads/posix/threads.c"
    "${source_dir}/src/time/posix/time.c")
else()
  set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
  find_package(Threads REQUIRED)
  target_link_libraries(ddsrt INTERFACE Threads::Threads)
  target_link_libraries(ddsrt INTERFACE ${CMAKE_DL_LIBS})

  set(DDSRT_HAVE_DYNLIB TRUE)
  set(DDSRT_HAVE_FILESYSTEM TRUE)

  list(APPEND sources
    "${source_dir}/src/heap/posix/heap.c")

  if(WIN32)
    set(DDSRT_HAVE_NETSTAT TRUE)
    set(DDSRT_HAVE_RUSAGE TRUE)

    list(APPEND headers
      "${source_dir}/include/dds/ddsrt/filesystem/windows.h"
      "${source_dir}/include/dds/ddsrt/sync/windows.h"
      "${source_dir}/include/dds/ddsrt/threads/windows.h"
      "${source_dir}/include/dds/ddsrt/types/windows.h")
    list(APPEND sources
      "${source_dir}/src/dynlib/windows/dynlib.c"
      "${source_dir}/src/environ/windows/environ.c"
      "${source_dir}/src/filesystem/windows/filesystem.c"
      "${source_dir}/src/netstat/windows/netstat.c"
      "${source_dir}/src/process/windows/process.c"
      "${source_dir}/src/random/windows/random.c"
      "${source_dir}/src/rusage/windows/rusage.c"
      "${source_dir}/src/sync/windows/sync.c"
      "${source_dir}/src/threads/windows/threads.c"
      "${source_dir}/src/time/windows/time.c")

    target_link_libraries(ddsrt INTERFACE ws2_32 iphlpapi bcrypt)
  else()
    list(APPEND headers
      "${source_dir}/include/dds/ddsrt/filesystem/posix.h"
      "${source_dir}/include/dds/ddsrt/sync/posix.h"
      "${source_dir}/include/dds/ddsrt/threads/posix.h"
      "${source_dir}/include/dds/ddsrt/types/posix.h")
    list(APPEND sources
      "${source_dir}/src/dynlib/posix/dynlib.c"
      "${source_dir}/src/environ/posix/environ.c"
      "${source_dir}/src/filesystem/posix/filesystem.c"
      "${source_dir}/src/process/posix/process.c"
      "${source_dir}/src/random/posix/random.c"
      "${source_dir}/src/sync/posix/sync.c"
      "${source_dir}/src/threads/posix/threads.c")

    if(CMAKE_SYSTEM MATCHES "Linux")
      set(DDSRT_HAVE_NETSTAT TRUE)
      set(DDSRT_HAVE_RUSAGE TRUE)
      list(APPEND sources
        "${source_dir}/src/netstat/linux/netstat.c"
        "${source_dir}/src/rusage/posix/rusage.c"
        "${source_dir}/src/time/posix/time.c")
    elseif(CMAKE_SYSTEM MATCHES "Darwin")
      set(DDSRT_HAVE_NETSTAT TRUE)
      set(DDSRT_HAVE_RUSAGE TRUE)
      list(APPEND sources
        "${source_dir}/src/netstat/darwin/netstat.c"
        "${source_dir}/src/rusage/posix/rusage.c"
        "${source_dir}/src/time/darwin/time.c")
    elseif(CMAKE_SYSTEM MATCHES "QNX")
      set(DDSRT_HAVE_NETSTAT TRUE)
      set(DDSRT_HAVE_RUSAGE TRUE)
      list(APPEND sources
        "${source_dir}/src/netstat/qnx/netstat.c"
        "${source_dir}/src/rusage/posix/rusage.c"
        "${source_dir}/src/time/posix/time.c")
    else()
      list(APPEND sources
        "${source_dir}/src/time/posix/time.c")
    endif()

    check_library_exists(c clock_gettime "" HAVE_CLOCK_GETTIME)
    if(NOT HAVE_CLOCK_GETTIME)
      # Before glibc 2.17, clock_gettime was in librt.
      check_library_exists(rt clock_gettime "time.h" HAVE_CLOCK_GETTIME_RT)
      if(HAVE_CLOCK_GETTIME_RT)
        set(HAVE_CLOCK_GETTIME TRUE)
        target_link_libraries(ddsrt INTERFACE rt)
      endif()
    endif()

    if(CMAKE_C_COMPILER_ID STREQUAL "SunPro")
      target_link_libraries(ddsrt INTERFACE socket nsl)
    elseif(CMAKE_SYSTEM_NAME MATCHES QNX)
      target_link_libraries(ddsrt INTERFACE socket m)
    endif()
  endif()
endif()

# Generate configuration file
set(DDSRT_WITH_LWIP ${WITH_LWIP})
set(DDSRT_WITH_FREERTOS ${WITH_FREERTOS})

foreach(feature TCP_TLS SECURITY LIFESPAN DEADLINE_MISSED NETWORK_PARTITIONS
                TYPELIB TYPE_DISCOVERY TOPIC_DISCOVERY QOS_PROVIDER)
  set(DDS_HAS_${feature} ${ENABLE_${feature}})
endforeach()

if(BUILD_TESTING)
  set(DDS_ALLOW_NESTED_DOMAIN 1)
endif()

if (NOT BUILD_SHARED_LIBS)
  set(DDS_IS_STATIC_LIBRARY 1)
endif()

configure_file(include/dds/config.h.in include/dds/config.h)
configure_file(include/dds/features.h.in include/dds/features.h)
configure_file(include/dds/version.h.in include/dds/version.h)

list(APPEND headers
  "${binary_dir}/include/dds/config.h"
  "${binary_dir}/include/dds/features.h"
  "${binary_dir}/include/dds/version.h")

target_sources(ddsrt INTERFACE ${headers} ${sources})

# Add library for internal use, i.e. for ddsconf and cunit_ddsrt, to avoid
# having to compile the same sources twice.
add_library(ddsrt-internal STATIC)
generate_export_header(
  ddsrt-internal BASE_NAME DDS EXPORT_FILE_NAME ddsrt-internal/include/dds/export.h)
target_link_libraries(
  ddsrt-internal PUBLIC $<TARGET_PROPERTY:ddsrt,INTERFACE_LINK_LIBRARIES>)
target_include_directories(
  ddsrt-internal PUBLIC ${binary_dir}/ddsrt-internal/include
                        $<TARGET_PROPERTY:ddsrt,INTERFACE_INCLUDE_DIRECTORIES>)
target_sources(
  ddsrt-internal PRIVATE $<TARGET_PROPERTY:ddsrt,INTERFACE_SOURCES>)

if(BUILD_TESTING)
  add_subdirectory(tests)
endif()

install(
  DIRECTORY
      "${source_dir}/include/"
      "${binary_dir}/include/"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
  COMPONENT dev
  FILES_MATCHING PATTERN "*.h")
