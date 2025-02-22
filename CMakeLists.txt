cmake_minimum_required(VERSION 3.18)

# Avoid warning about DOWNLOAD_EXTRACT_TIMESTAMP in CMake 3.24:
if(CMAKE_VERSION VERSION_GREATER_EQUAL "3.24.0")
  cmake_policy(SET CMP0135 NEW)
endif()

set(CMAKE_CXX_STANDARD 17)
project(pika)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
enable_testing()

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
  # using Clang
  if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "5.0")
    message(FATAL_ERROR "Clang version must be greater than 5.0")
  endif()
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  # using GCC
  if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "7.0")
    message(FATAL_ERROR "GCC G++ version must be greater than 7.0")
  endif()
endif()


############# You should enable sanitizer if you are developing pika #############
# Uncomment the following two lines to enable AddressSanitizer to detect memory leaks and other memory-related bugs.
#set(CMAKE_BUILD_TYPE "Debug")
#set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=address  -O0 -fno-omit-frame-pointer -fno-optimize-sibling-calls")

# [Notice] AddressSanitizer and ThreadSanitizer can not be enabled at the same time.

# Uncomment the following two lines to enable ThreadSanitizer to detect data race and other thread-related issue.
#set(CMAKE_BUILD_TYPE "Debug")
#set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fsanitize=thread -O0 -fno-omit-frame-pointer -fno-optimize-sibling-calls")

execute_process(COMMAND uname -p OUTPUT_VARIABLE HOST_ARCH)
string(TOLOWER ${HOST_ARCH} HOST_ARCH)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE RELEASE)
endif()

string(TOUPPER ${CMAKE_BUILD_TYPE} BUILD_TYPE)

if(${BUILD_TYPE} STREQUAL DEBUG)
  set(LIB_BUILD_TYPE DEBUG)
elseif(${BUILD_TYPE} STREQUAL MINSIZEREL)
  set(LIB_BUILD_TYPE MINSIZEREL)
elseif(${BUILD_TYPE} STREQUAL RELWITHDEBINFO)
  set(LIB_BUILD_TYPE RELWITHDEBINFO)
else()
  set(LIB_BUILD_TYPE RELEASE)
  set(CMAKE_CXX_FLAGS_RELEASE "-O2 -DNDEBUG")
endif()

if(CMAKE_SYSTEM_NAME MATCHES "Darwin")
   set(CMAKE_CXX_FLAGS "-pthread")
   add_definitions(-DOS_MACOSX)
elseif(CMAKE_SYSTEM_NAME MATCHES "Linux")
   if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
     set(CMAKE_EXE_LINKER_FLAGS "-stdlib=libc++ -fuse-ld=lld -lc++ -lc++abi ${CMAKE_EXE_LINKER_FLAGS}")
     set(CMAKE_CXX_FLAGS "-stdlib=libc++ -pthread ${CMAKE_CXX_FLAGS}")
   elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
     set(CMAKE_EXE_LINKER_FLAGS "-static-libgcc -static-libstdc++")
     set(CMAKE_CXX_FLAGS "-pthread -Wl,--no-as-needed -ldl")
   endif()
   add_definitions(-DOS_LINUX)
else()
   message(FATAL_ERROR "only support linux or macOs")
endif()

if(HOST_ARCH MATCHES "x86_64" OR HOST_ARCH MATCHES "i386")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse -msse4.2")
elseif(HOST_ARCH MATCHES "arm")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=armv8-a+crc+crypto -moutline-atomics")
endif()

set(EP_BASE_SUFFIX "buildtrees")
set_property(DIRECTORY PROPERTY EP_BASE ${CMAKE_CURRENT_SOURCE_DIR}/${EP_BASE_SUFFIX})
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake/modules/")
set(STAGED_INSTALL_PREFIX ${CMAKE_CURRENT_SOURCE_DIR}/deps)
set(CMAKE_UTILS_DIR ${CMAKE_CURRENT_SOURCE_DIR}/utils)
set(INSTALL_INCLUDEDIR ${STAGED_INSTALL_PREFIX}/include)
set(INSTALL_LIBDIR ${STAGED_INSTALL_PREFIX}/lib)
set(INSTALL_LIBDIR_64 ${STAGED_INSTALL_PREFIX}/lib64)
set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} ${STAGED_INSTALL_PREFIX})

execute_process(COMMAND sh ${CMAKE_UTILS_DIR}/Get_OS_Version.sh
  OUTPUT_VARIABLE OS_VERSION)

message(STATUS "${PROJECT_NAME} staged install: ${STAGED_INSTALL_PREFIX}")
message(STATUS "Current platform: ${OS_VERSION} ")
cmake_host_system_information(RESULT CPU_CORE QUERY NUMBER_OF_LOGICAL_CORES)
message(STATUS "Cpu core ${CPU_CORE}")

find_program(AUTOCONF
        autoconf
        PATHS /usr/bin /usr/local/bin)

if (${AUTOCONF} MATCHES AUTOCONF-NOTFOUND)
  message(FATAL_ERROR "not find autoconf on localhost")
endif()

#set(CLANG_SEARCH_PATH "/usr/local/bin" "/usr/bin" "/usr/local/opt/llvm/bin"
#                      "/usr/local/opt/llvm@12/bin")
find_program(CLANG_TIDY_BIN
        NAMES clang-tidy clang-tidy-12
        HINTS ${CLANG_SEARCH_PATH})
if ("${CLANG_TIDY_BIN}" STREQUAL "CLANG_TIDY_BIN-NOTFOUND")
  message(WARNING "couldn't find clang-tidy.")
else ()
  message(STATUS "found clang-tidy at ${CLANG_TIDY_BIN}")
endif ()

find_program(CLANG_APPLY_REPLACEMENTS_BIN
        NAMES clang-apply-replacements clang-apply-replacements-12
        HINTS ${CLANG_SEARCH_PATH})

if ("${CLANG_APPLY_REPLACEMENTS_BIN}" STREQUAL "CLANG_APPLY_REPLACEMENTS_BIN-NOTFOUND")
    message(WARNING "couldn't find clang-apply-replacements.")
else ()
    message(STATUS "found clang-apply-replacements at ${CLANG_APPLY_REPLACEMENTS_BIN}")
endif ()

option(WITH_COMMAND_DOCS "build with command docs support" OFF)
if (WITH_COMMAND_DOCS)
  add_definitions(-DWITH_COMMAND_DOCS)
endif()

include(protogen.cmake)
include(ExternalProject)

ExternalProject_Add(gtest
  URL
  https://github.com/google/googletest/archive/refs/tags/release-1.12.1.tar.gz
  URL_HASH
  MD5=e82199374acdfda3f425331028eb4e2a
  DOWNLOAD_NO_PROGRESS
  1
  UPDATE_COMMAND
  ""
  LOG_CONFIGURE
  1
  LOG_BUILD
  1
  LOG_INSTALL
  1
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
  -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
  BUILD_ALWAYS
  1
  BUILD_COMMAND
  make -j${CPU_CORE}
)

if(${OS_VERSION} MATCHES "CentOS")
  set(GTEST_LIBRARY ${INSTALL_LIBDIR_64}/libgtest.a)
  set(GTEST_MAIN_LIBRARY ${INSTALL_LIBDIR_64}/libgtest_main.a)
  set(GMOCK_LIBRARY ${INSTALL_LIBDIR_64}/libgmock.a)
else()
  set(GTEST_LIBRARY ${INSTALL_LIBDIR}/libgtest.a)
  set(GTEST_MAIN_LIBRARY ${INSTALL_LIBDIR}/libgtest_main.a)
  set(GMOCK_LIBRARY ${INSTALL_LIBDIR}/libgmock.a)
endif()

set(GTEST_INCLUDE_DIR ${INSTALL_INCLUDEDIR})
set(GTEST_MAIN_INCLUDE_DIR ${INSTALL_INCLUDEDIR})
set(GMOCK_INCLUDE_DIR ${INSTALL_INCLUDEDIR})


ExternalProject_Add(gflags
  URL
  https://github.com/gflags/gflags/archive/refs/tags/v2.2.2.tar.gz
  URL_HASH
  MD5=1a865b93bacfa963201af3f75b7bd64c
  DOWNLOAD_NO_PROGRESS
  1
  UPDATE_COMMAND
  ""
  LOG_CONFIGURE
  1
  LOG_BUILD
  1
  LOG_INSTALL
  1
  BUILD_ALWAYS
  1
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
  -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
  -DGFLAGS_NAMESPACE=gflags
  -DBUILD_STATIC_LIBS=ON
  -DBUILD_SHARED_LIBS=OFF
  BUILD_COMMAND
  make -j${CPU_CORE}
)

if(${LIB_BUILD_TYPE} STREQUAL DEBUG)
  set(LIB_GFLAGS libgflags_debug.a)
else()
  set(LIB_GFLAGS libgflags.a)
endif()

set(GFLAGS_LIBRARY ${INSTALL_LIBDIR}/${LIB_GFLAGS})
set(GFLAGS_INCLUDE_DIR ${INSTALL_INCLUDEDIR})

if(CMAKE_SYSTEM_NAME MATCHES "Linux")
  ExternalProject_Add(unwind
    DEPENDS
    URL
    https://github.com/libunwind/libunwind/releases/download/v1.6.2/libunwind-1.6.2.tar.gz
    URL_HASH
    MD5=f625b6a98ac1976116c71708a73dc44a
    DOWNLOAD_NO_PROGRESS
    1
    UPDATE_COMMAND
    ""
    LOG_CONFIGURE
    1
    LOG_BUILD
    1
    LOG_INSTALL
    1
    CONFIGURE_COMMAND
    <SOURCE_DIR>/configure --prefix=${STAGED_INSTALL_PREFIX} --enable-minidebuginfo=no --enable-zlibdebuginfo=no --enable-shared=no
    BUILD_IN_SOURCE
    1
    BUILD_COMMAND
    make -j${CPU_CORE}
    INSTALL_COMMAND
    make install
  )
  set(LIBUNWIND_LIBRARY ${INSTALL_LIBDIR}/libunwind.a)
  set(LIBUNWIND_INCLUDE_DIR ${INSTALL_INCLUDEDIR})
  set(LIBUNWIND_NAME unwind)
  set(LIBUNWIND_ON ON)
else()
  set(LIBUNWIND_ON OFF)
endif()

ExternalProject_Add(glog
  DEPENDS
  gflags
  gtest
  ${LIBUNWIND_NAME}
  URL
  https://github.com/google/glog/archive/refs/tags/v0.6.0.tar.gz
  URL_HASH
  MD5=c98a6068bc9b8ad9cebaca625ca73aa2
  DOWNLOAD_NO_PROGRESS
  1
  UPDATE_COMMAND
  ""
  LOG_CONFIGURE
  1
  LOG_BUILD
  1
  LOG_INSTALL
  1
  BUILD_ALWAYS
  1
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
  -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
  -DWITH_GFLAGS=ON
  -DBUILD_TESTING=OFF
  -DBUILD_SHARED_LIBS=OFF
  -DWITH_UNWIND=${LIBUNWIND_ON}
  -DCMAKE_PREFIX_PATH=${CMAKE_PREFIX_PATH}
  BUILD_COMMAND
  make -j${CPU_CORE}
)

if(${LIB_BUILD_TYPE} STREQUAL DEBUG)
  set(LIB_GLOG libglogd.a)
else()
  set(LIB_GLOG libglog.a)
endif()

if(${OS_VERSION} MATCHES "CentOS")
  set(GLOG_LIBRARY ${INSTALL_LIBDIR_64}/${LIB_GLOG})
else()
  set(GLOG_LIBRARY ${INSTALL_LIBDIR}/${LIB_GLOG})
endif()
set(GLOG_INCLUDE_DIR ${INSTALL_INCLUDEDIR})

ExternalProject_Add(snappy
  DEPENDS
  URL
  https://github.com/google/snappy/archive/refs/tags/1.1.7.tar.gz
  URL_HASH
  MD5=ee9086291c9ae8deb4dac5e0b85bf54a
  DOWNLOAD_NO_PROGRESS
  1
  UPDATE_COMMAND
  ""
  LOG_CONFIGURE
  1
  LOG_BUILD
  1
  LOG_INSTALL
  1
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
  -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
  -DSNAPPY_BUILD_TESTS=OFF
  -DBUILD_STATIC_LIBS=ON
  -DBUILD_SHARED_LIBS=OFF
  BUILD_ALWAYS
  1
  BUILD_COMMAND
  make -j${CPU_CORE}
)

if(${OS_VERSION} MATCHES "CentOS")
  set(SNAPPY_LIBRARY ${INSTALL_LIBDIR_64}/libsnappy.a)
else()
  set(SNAPPY_LIBRARY ${INSTALL_LIBDIR}/libsnappy.a)
endif()

set(SNAPPY_INCLUDE_DIR ${INSTALL_INCLUDEDIR})

ExternalProject_Add(zstd
  DEPENDS
  URL
  https://github.com/facebook/zstd/releases/download/v1.5.4/zstd-1.5.4.tar.gz
  URL_HASH
  MD5=2352b1f9ccc7446641046bb3d440c3ed
  DOWNLOAD_NO_PROGRESS
  1
  UPDATE_COMMAND
  ""
  LOG_CONFIGURE
  1
  LOG_BUILD
  1
  LOG_INSTALL
  1
  SOURCE_SUBDIR
  build/cmake
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
  -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
  -DBUILD_TESTING=OFF
  -DZSTD_BUILD_STATIC=ON
  -DZSTD_BUILD_SHARED=OFF
  BUILD_ALWAYS
  1
  BUILD_COMMAND
  make -j${CPU_CORE}
)

if(${OS_VERSION} MATCHES "CentOS")
  set(ZSTD_LIBRARY ${INSTALL_LIBDIR_64}/libzstd.a)
else()
  set(ZSTD_LIBRARY ${INSTALL_LIBDIR}/libzstd.a)
endif()

set(ZSTD_INCLUDE_DIR ${INSTALL_INCLUDEDIR})

ExternalProject_Add(fmt
  DEPENDS
  URL
  https://github.com/fmtlib/fmt/archive/refs/tags/7.1.0.tar.gz
  URL_HASH
  MD5=32af902636d373641f4ef9032fc65b3a
  DOWNLOAD_NO_PROGRESS
  1
  UPDATE_COMMAND
  ""
  LOG_CONFIGURE
  1
  LOG_BUILD
  1
  LOG_INSTALL
  1
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
  -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
  BUILD_ALWAYS
  1
  BUILD_COMMAND
  make -j${CPU_CORE}
)

if(${LIB_BUILD_TYPE} STREQUAL DEBUG)
  set(LIB_FMT libfmtd.a)
else()
  set(LIB_FMT libfmt.a)
endif()

if(${OS_VERSION} MATCHES "CentOS")
  set(FMT_LIBRARY ${INSTALL_LIBDIR_64}/${LIB_FMT})
else()
  set(FMT_LIBRARY ${INSTALL_LIBDIR}/${LIB_FMT})
endif()

set(FMT_INCLUDE_DIR ${INSTALL_INCLUDEDIR})

ExternalProject_Add(lz4
  DEPENDS
  URL
  https://github.com/lz4/lz4/archive/refs/tags/v1.9.4.tar.gz
  URL_HASH
  MD5=e9286adb64040071c5e23498bf753261
  DOWNLOAD_NO_PROGRESS
  1
  UPDATE_COMMAND
  ""
  LOG_CONFIGURE
  1
  LOG_BUILD
  1
  LOG_INSTALL
  1
  SOURCE_SUBDIR
  build/cmake
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
  -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
  -DBUILD_TESTING=OFF
  -DBUILD_STATIC_LIBS=ON
  -DBUILD_SHARED_LIBS=OFF
  BUILD_ALWAYS
  1
  BUILD_COMMAND
  make -j${CPU_CORE}
)

if(${OS_VERSION} MATCHES "CentOS")
  set(LZ4_LIBRARY ${INSTALL_LIBDIR_64}/liblz4.a)
else()
  set(LZ4_LIBRARY ${INSTALL_LIBDIR}/liblz4.a)
endif()

set(LZ4_INCLUDE_DIR ${INSTALL_INCLUDEDIR})

ExternalProject_Add(zlib
  DEPENDS
  URL
  https://github.com/madler/zlib/releases/download/v1.2.13/zlib-1.2.13.tar.gz
  URL_HASH
  MD5=9b8aa094c4e5765dabf4da391f00d15c
  DOWNLOAD_NO_PROGRESS
  1
  UPDATE_COMMAND
  ""
  LOG_CONFIGURE
  1
  LOG_BUILD
  1
  LOG_INSTALL
  1
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
  -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
  BUILD_ALWAYS
  1
  BUILD_COMMAND
  make -j${CPU_CORE}
)

set(ZLIB_LIBRARY ${INSTALL_LIBDIR}/libz.a)
set(ZLIB_INCLUDE_DIR ${INSTALL_INCLUDEDIR})

if(CMAKE_SYSTEM_NAME MATCHES "Linux")
  ExternalProject_Add(gperftools
    DEPENDS
    unwind
    URL
    https://github.com/gperftools/gperftools/releases/download/gperftools-2.10/gperftools-2.10.tar.gz
    URL_HASH
    MD5=62bf6c76ba855ed580de5e139bd2a483
    DOWNLOAD_NO_PROGRESS
    1
    UPDATE_COMMAND
    ""
    LOG_CONFIGURE
    1
    LOG_BUILD
    1
    LOG_INSTALL
    1
    CMAKE_ARGS
    -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
    -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
    -DGPERFTOOLS_BUILD_STATIC=ON
    -DEFAULT_BUILD_MINIMAL=ON
    -Dgperftools_build_benchmark=OFF
    BUILD_COMMAND
    make -j${CPU_CORE}
  )
  set(LIBGPERF_NAME gperftools)
endif()

if(CMAKE_SYSTEM_NAME MATCHES "Linux")
  ExternalProject_Add(jemalloc
    DEPENDS
    URL
    https://github.com/jemalloc/jemalloc/archive/refs/tags/5.3.0.tar.gz
    URL_HASH
    MD5=594dd8e0a1e8c1ef8a1b210a1a5aff5b
    DOWNLOAD_NO_PROGRESS
    1
    UPDATE_COMMAND
    ""
    LOG_CONFIGURE
    1
    LOG_BUILD
    1
    LOG_INSTALL
    1
    CONFIGURE_COMMAND
    <SOURCE_DIR>/autogen.sh --prefix=${STAGED_INSTALL_PREFIX}
    BUILD_IN_SOURCE
    1
    BUILD_COMMAND
    make -j${CPU_CORE}
    BUILD_ALWAYS
    1
    INSTALL_COMMAND
    make install
  )

  set(JEMALLOC_LIBRARY ${INSTALL_LIBDIR}/libjemalloc.a)
  set(JEMALLOC_INCLUDE_DIR ${INSTALL_INCLUDEDIR})
  set(LIBJEMALLOC_NAME jemalloc)
  set(JEMALLOC_ON ON)
else()
  set(JEMALLOC_ON OFF)
endif()

ExternalProject_Add(protobuf
  DEPENDS
  zlib
  URL
  https://github.com/protocolbuffers/protobuf/releases/download/v3.17.3/protobuf-cpp-3.17.3.tar.gz
  URL_HASH
  MD5=3fe4c2647e0991c014a386a896d0a116
  DOWNLOAD_NO_PROGRESS
  1
  UPDATE_COMMAND
  ""
  LOG_CONFIGURE
  1
  LOG_BUILD
  1
  LOG_INSTALL
  1
  SOURCE_SUBDIR
  cmake
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
  -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
  -DBUILD_SHARED_LIBS=FALSE
  -Dprotobuf_BUILD_TESTS=FALSE
  BUILD_IN_SOURCE
  1
  BUILD_ALWAYS
  1
  BUILD_COMMAND
  make -j${CPU_CORE}
)

if(${LIB_BUILD_TYPE} STREQUAL DEBUG)
  set(LIB_PROTOBUF libprotobufd.a)
else()
  set(LIB_PROTOBUF libprotobuf.a)
endif()

set(PROTOBUF_INCLUDE_DIR ${INSTALL_INCLUDEDIR})
set(PROTOBUF_LIBRARY ${INSTALL_LIBDIR}/${LIB_PROTOBUF})
set(PROTOBUF_PROTOC ${STAGED_INSTALL_PREFIX}/bin/protoc)

ExternalProject_Add(rocksdb
  DEPENDS
  gflags
  gtest
  snappy
  zstd
  lz4
  zlib
  ${LIBGPERF_NAME}
  ${LIBJEMALLOC_NAME}
  URL
  https://github.com/facebook/rocksdb/archive/refs/tags/v8.3.3.tar.gz
  URL_HASH
  MD5=2815555a2ee2d12a7081a91fb988bf24
  DOWNLOAD_NO_PROGRESS
  1
  UPDATE_COMMAND
  ""
  LOG_CONFIGURE
  1
  LOG_BUILD
  1
  LOG_INSTALL
  1
  BUILD_ALWAYS
  1
  CMAKE_ARGS
  -DCMAKE_INSTALL_PREFIX=${STAGED_INSTALL_PREFIX}
  -DCMAKE_PREFIX_PATH=${CMAKE_PREFIX_PATH}
  -DCMAKE_BUILD_TYPE=${LIB_BUILD_TYPE}
  -DUSE_RTTI=1
  -DWITH_BENCHMARK=OFF
  -DWITH_BENCHMARK_TOOLS=OFF
  -DWITH_TOOLS=OFF
  -DWITH_CORE_TOOLS=OFF
  -DWITH_TESTS=OFF
  -DWITH_TRACE_TOOLS=OFF
  -DWITH_EXAMPLES=OFF
  -DROCKSDB_BUILD_SHARED=OFF
  -DWITH_JEMALLOC=${JEMALLOC_ON}
  -DWITH_LZ4=ON
  -DWITH_SNAPPY=ON
  -DWITH_ZLIB=ON
  -DWITH_ZSTD=ON
  -DWITH_GFLAGS=ON
  -DFAIL_ON_WARNINGS=OFF
  -DWITH_LIBURING=OFF
  -DPORTABLE=1
  BUILD_COMMAND
  make -j${CPU_CORE}
)

option(USE_PIKA_TOOLS "compile pika-tools" OFF)
if (USE_PIKA_TOOLS)
  ExternalProject_Add(hiredis
    URL
    https://github.com/vipshop/hiredis-vip/archive/refs/tags/0.3.0.tar.gz
    URL_HASH
    MD5=e4d4a2928188b11f696d35ca1c52e9a4
    DOWNLOAD_NO_PROGRESS
    1
    UPDATE_COMMAND
    ""
    LOG_BUILD
    1
    LOG_INSTALL
    1
    BUILD_IN_SOURCE
    1
    SOURCE_SUBDIR
    ""
    BUILD_ALWAYS
    1
    CONFIGURE_COMMAND
    ""
    BUILD_COMMAND
    make PREFIX=${STAGED_INSTALL_PREFIX} -j${CPU_CORE} all
    INSTALL_COMMAND
    make PREFIX=${STAGED_INSTALL_PREFIX} -j${CPU_CORE} install
  )
  set(HIREDIS_LIBRARY ${INSTALL_LIBDIR}/libhiredis_vip.a)

  ExternalProject_Add(bz2
    URL
    https://sourceware.org/pub/bzip2/bzip2-1.0.8.tar.gz
    URL_HASH
    MD5=67e051268d0c475ea773822f7500d0e5
    DOWNLOAD_NO_PROGRESS
    1
    UPDATE_COMMAND
    ""
    LOG_BUILD
    1
    LOG_INSTALL
    1
    BUILD_IN_SOURCE
    1
    SOURCE_SUBDIR
    ""
    BUILD_ALWAYS
    1
    CONFIGURE_COMMAND
    ""
    BUILD_COMMAND
    make PREFIX=${STAGED_INSTALL_PREFIX} -j${CPU_CORE} all
    INSTALL_COMMAND
    make PREFIX=${STAGED_INSTALL_PREFIX} -j${CPU_CORE} install
  )
  set(BZ2_LIBRARY ${INSTALL_LIBDIR}/libbz2.a)
endif()

if(${OS_VERSION} MATCHES "CentOS")
  set(ROCKSDB_LIBRARY ${INSTALL_LIBDIR_64}/librocksdb.a)
else()
  set(ROCKSDB_LIBRARY ${INSTALL_LIBDIR}/librocksdb.a)
endif()

set(ROCKSDB_INCLUDE_DIR ${INSTALL_INCLUDEDIR})
set(ROCKSDB_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/${EP_BASE_SUFFIX}/Source/rocksdb)

add_subdirectory(src/pstd)
add_subdirectory(src/net)
add_subdirectory(src/storage)
if (USE_PIKA_TOOLS)
  add_subdirectory(tools)
endif()
aux_source_directory(src DIR_SRCS)

# # generate version
string(TIMESTAMP TS "%Y-%m-%d %H:%M:%S" UTC)
set(PIKA_BUILD_DATE "${TS}" CACHE STRING "the time we first built pika")

find_package(Git)

if(GIT_FOUND AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git")
  execute_process(WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" OUTPUT_VARIABLE PIKA_GIT_SHA COMMAND "${GIT_EXECUTABLE}" rev-parse HEAD)
  execute_process(WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" RESULT_VARIABLE PIKA_GIT_MOD COMMAND "${GIT_EXECUTABLE}" diff-index HEAD --quiet)
  execute_process(WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" OUTPUT_VARIABLE PIKA_GIT_DATE COMMAND "${GIT_EXECUTABLE}" log -1 --date=format:"%Y-%m-%d %T" --format="%ad")
  execute_process(WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" OUTPUT_VARIABLE PIKA_GIT_TAG RESULT_VARIABLE rv COMMAND "${GIT_EXECUTABLE}" symbolic-ref -q --short HEAD OUTPUT_STRIP_TRAILING_WHITESPACE)

  if(rv AND NOT rv EQUAL 0)
    execute_process(WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}" OUTPUT_VARIABLE PIKA_GIT_TAG COMMAND "${GIT_EXECUTABLE}" describe --tags --exact-match OUTPUT_STRIP_TRAILING_WHITESPACE)
  endif()
else()
  set(PIKA_GIT_SHA 0)
  set(PIKA_GIT_MOD 1)
endif()

string(REGEX REPLACE "[^0-9a-fA-F]+" "" PIKA_GIT_SHA "${PIKA_GIT_SHA}")
string(REGEX REPLACE "[^0-9: /-]+" "" PIKA_GIT_DATE "${PIKA_GIT_DATE}")

message("pika GIT_SHA = ${PIKA_GIT_SHA}")
message("pika GIT_MOD = ${PIKA_GIT_MOD}")
message("pika GIT_DATE = ${PIKA_GIT_DATE}")
message("pika GIT_TAG = ${PIKA_GIT_TAG}")
message("pika BUILD_DATE = ${PIKA_BUILD_DATE}")

set(PIKA_BUILD_VERSION_CC ${CMAKE_BINARY_DIR}/pika_build_version.cc)
message("PIKA_BUILD_VERSION_CC : " ${PIKA_BUILD_VERSION_CC})
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/build_version.cc.in ${PIKA_BUILD_VERSION_CC} @ONLY)

set(PROTO_FILES ${CMAKE_CURRENT_SOURCE_DIR}/src/pika_inner_message.proto ${CMAKE_CURRENT_SOURCE_DIR}/src/rsync_service.proto)
custom_protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS ${PROTO_FILES})
message("pika PROTO_SRCS = ${PROTO_SRCS}")
message("pika PROTO_HDRS = ${PROTO_HDRS}")

add_executable(${PROJECT_NAME}
  ${DIR_SRCS}
  ${PROTO_SRCS}
  ${PROTO_HDRS}
  ${PIKA_BUILD_VERSION_CC})

target_link_directories(${PROJECT_NAME}
  PUBLIC ${INSTALL_LIBDIR_64}
  PUBLIC ${INSTALL_LIBDIR})

add_dependencies(${PROJECT_NAME}
  gflags
  gtest
  ${LIBUNWIND_NAME}
  glog
  fmt
  snappy
  zstd
  lz4
  zlib
  ${LIBGPERF_NAME}
  ${LIBJEMALLOC_NAME}
  rocksdb
  protobuf
  pstd
  net
  storage)

target_include_directories(${PROJECT_NAME}
  PUBLIC ${CMAKE_CURRENT_BINARY_DIR}
  PUBLIC ${PROJECT_SOURCE_DIR}
  ${INSTALL_INCLUDEDIR}
)

target_link_libraries(${PROJECT_NAME}
  storage
  net
  pstd
  ${GLOG_LIBRARY}
  librocksdb.a
  ${LIB_PROTOBUF}
  ${LIB_GFLAGS}
  ${LIB_FMT}
  libsnappy.a
  libzstd.a
  liblz4.a
  libz.a
  ${LIBUNWIND_LIBRARY}
  ${JEMALLOC_LIBRARY})

option(USE_SSL "Enable SSL support" OFF)
add_custom_target(
        clang-tidy
        COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/run_clang_tidy.py
        -clang-tidy-binary ${CLANG_TIDY_BIN}
        -p ${CMAKE_BINARY_DIR}
        -quiet
        -extra-arg=-std=c++17
    )

add_custom_target(clang-tidy-fix
    ${CMAKE_CURRENT_SOURCE_DIR}/run_clang_tidy.py
    -clang-tidy-binary ${CLANG_TIDY_BIN}
    -p ${CMAKE_BINARY_DIR}
    -clang-apply-replacements-binary ${CLANG_APPLY_REPLACEMENTS_BIN}
    -fix
    -extra-arg=-std=c++17
)
