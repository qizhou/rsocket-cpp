cmake_minimum_required(VERSION 3.2)

# The RSOCKET_CC CMake variable specifies the C compiler, e.g. gcc-4.9.
# The C++ compiler name is obtained by replacing "gcc" with "g++" and "clang"
# with "clang++"". If RSOCKET_CC is not given, the compiler is detected
# automatically.
if (RSOCKET_CC)
  set(ENV{CC} ${RSOCKET_CC})
  if (${RSOCKET_CC} MATCHES clang)
    string(REPLACE clang clang++ CXX ${RSOCKET_CC})
  else ()
    string(REPLACE gcc g++ CXX ${RSOCKET_CC})
  endif ()
  set(ENV{CXX} ${CXX})
endif ()

project(ReactiveSocket)

# CMake modules.
set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/")

# Joins arguments and stores the result in ${var}.
function(join var)
  unset(result)
  foreach (arg ${ARGN})
    if (DEFINED result)
      set(result "${result}${arg}")
    else ()
      set(result "${arg}")
    endif ()
  endforeach ()
  set(${var} "${result}" PARENT_SCOPE)
endfunction()

# Generate compilation database for use by YouCompleteMe.
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

# make sure to bail on in-source builds for cleanliness
if (${CMAKE_SOURCE_DIR} STREQUAL ${CMAKE_BINARY_DIR})
  message(FATAL_ERROR "In-source builds not allowed. Please make a new directory (called a build directory)"
  "and run CMake from there. You may need to remove CMakeCache.txt.")
endif()

# default built type is Debug
if (NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Choose the type of build" FORCE)
endif(NOT CMAKE_BUILD_TYPE)

string(TOLOWER "${CMAKE_BUILD_TYPE}" BUILD_TYPE_LOWER)

# Enable ASAN by default on debug macOS builds.
if (APPLE)
  set(OPENSSL_ROOT_DIR "/usr/local/opt/openssl")
  if ("${BUILD_TYPE_LOWER}" MATCHES "debug")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address,integer -fno-sanitize=unsigned-integer-overflow")
  endif()
endif()

# Add compiler-specific options.
if (CMAKE_COMPILER_IS_GNUCXX)
  if (RSOCKET_ASAN)
    set(ASAN_FLAGS -fsanitize=address,undefined)
  endif ()
  set(EXTRA_LINK_FLAGS ${EXTRA_LINK_FLAGS} -fuse-ld=gold)

  # Enable code coverage.
  add_compile_options(--coverage)
  set(EXTRA_LINK_FLAGS ${EXTRA_LINK_FLAGS} --coverage)
  set(COVERAGE_INFO coverage.info)
  add_custom_command(
    OUTPUT ${COVERAGE_INFO}
    # Capture coverage info.
    COMMAND lcov --directory . --capture --output-file ${COVERAGE_INFO}
    # Filter out system and test code.
    COMMAND lcov --remove ${COVERAGE_INFO} 'tests/*' 'test/*' 'tck-test/*' '/usr/*' 'gmock/*' 'folly/*' --output-file
                 ${COVERAGE_INFO}
    # Debug before upload.
    COMMAND lcov --list ${COVERAGE_INFO})
elseif (${CMAKE_CXX_COMPILER_ID} MATCHES Clang)
  if (RSOCKET_ASAN)
    set(ASAN_FLAGS
      -fsanitize=address,undefined,integer
      -fno-sanitize=unsigned-integer-overflow)
  endif ()
endif ()

if (DEFINED ASAN_FLAGS)
  add_compile_options(${ASAN_FLAGS})
  set(EXTRA_LINK_FLAGS ${EXTRA_LINK_FLAGS} ${ASAN_FLAGS})
endif ()

add_custom_target(coverage DEPENDS ${COVERAGE_INFO})

option(BUILD_BENCHMARKS "Build benchmarks" ON)

enable_testing()

include(ExternalProject)
include(CTest)

if (NOT FOLLY_INSTALL_DIR)
  set(FOLLY_INSTALL_DIR $ENV{HOME}/folly)
endif ()

# Check if the correct version of folly is already installed.
set(FOLLY_VERSION v2017.07.10.00)
set(FOLLY_VERSION_FILE ${FOLLY_INSTALL_DIR}/${FOLLY_VERSION})
if (RSOCKET_INSTALL_DEPS)
  if (NOT EXISTS ${FOLLY_VERSION_FILE})
    # Remove the old version of folly.
    file(REMOVE_RECURSE ${FOLLY_INSTALL_DIR})
    set(INSTALL_FOLLY True)
  endif ()
endif ()

if (INSTALL_FOLLY)
  # Build and install folly.
  ExternalProject_Add(
    folly-ext
    GIT_REPOSITORY https://github.com/facebook/folly
    GIT_TAG ${FOLLY_VERSION}
    BINARY_DIR folly-ext-prefix/src/folly-ext/folly
    CONFIGURE_COMMAND autoreconf -ivf
      COMMAND ./configure CXX=${CMAKE_CXX_COMPILER}
                          --prefix=${FOLLY_INSTALL_DIR}
    BUILD_COMMAND make -j4
    INSTALL_COMMAND make install
      COMMAND cmake -E touch ${FOLLY_VERSION_FILE})

  set(FOLLY_INCLUDE_DIR ${FOLLY_INSTALL_DIR}/include)
  set(lib ${CMAKE_SHARED_LIBRARY_PREFIX}folly${CMAKE_SHARED_LIBRARY_SUFFIX})
  set(FOLLY_LIBRARY ${FOLLY_INSTALL_DIR}/lib/${lib})

  # CMake requires directories listed in INTERFACE_INCLUDE_DIRECTORIES to exist.
  file(MAKE_DIRECTORY ${FOLLY_INCLUDE_DIR})
else ()
  # Use installed folly.
  find_package(Folly REQUIRED)
endif ()

find_package(Threads)
find_library(EVENT_LIBRARY event)

add_library(folly SHARED IMPORTED)
set_property(TARGET folly PROPERTY IMPORTED_LOCATION ${FOLLY_LIBRARY})
set_property(TARGET folly
  APPEND PROPERTY INTERFACE_LINK_LIBRARIES
  ${EXTRA_LINK_FLAGS} ${EVENT_LIBRARY} ${CMAKE_THREAD_LIBS_INIT})
if (TARGET folly-ext)
  add_dependencies(folly folly-ext)
endif ()

# Folly includes are marked as system to prevent errors on non-standard
# extensions when compiling with -pedantic and -Werror.
set_property(TARGET folly
  APPEND PROPERTY INTERFACE_SYSTEM_INCLUDE_DIRECTORIES ${FOLLY_INCLUDE_DIR})
set_property(TARGET folly
  APPEND PROPERTY INTERFACE_INCLUDE_DIRECTORIES ${FOLLY_INCLUDE_DIR})

# gmock
ExternalProject_Add(
  gmock
  URL ${CMAKE_CURRENT_SOURCE_DIR}/googletest-release-1.8.0.zip
  INSTALL_COMMAND ""
)

ExternalProject_Get_Property(gmock source_dir)
set(GMOCK_SOURCE_DIR ${source_dir})
ExternalProject_Get_Property(gmock binary_dir)
set(GMOCK_BINARY_DIR ${binary_dir})

set(GMOCK_LIBS
  ${GMOCK_BINARY_DIR}/${CMAKE_CFG_INTDIR}/googlemock/${CMAKE_STATIC_LIBRARY_PREFIX}gmock${CMAKE_STATIC_LIBRARY_SUFFIX}
  ${GMOCK_BINARY_DIR}/${CMAKE_CFG_INTDIR}/googlemock/${CMAKE_STATIC_LIBRARY_PREFIX}gmock_main${CMAKE_STATIC_LIBRARY_SUFFIX}
  )

set(CMAKE_CXX_STANDARD 14)

# Common configuration for all build modes.
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -pedantic")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Woverloaded-virtual")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")

set(EXTRA_CXX_FLAGS ${EXTRA_CXX_FLAGS} -Werror)

if("${BUILD_TYPE_LOWER}" MATCHES "debug")
  message("debug mode was set")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-unreachable-code")
else()
  message("release mode was set")
endif()

if("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
  set(TEST_CXX_FLAGS ${TEST_CXX_FLAGS} -Wno-inconsistent-missing-override)
endif()

find_library(DOUBLE-CONVERSION double-conversion)

find_package(OpenSSL REQUIRED)

# Find glog and gflags libraries specifically
find_path(GLOG_INCLUDE_DIR glog/logging.h)
find_path(GFLAGS_INCLUDE_DIR gflags/gflags.h)

find_library(GLOG_LIBRARY glog)
find_library(GFLAGS_LIBRARY gflags)

message("gflags include_dir <${GFLAGS_INCLUDE_DIR}> lib <${GFLAGS_LIBRARY}>")
message("glog include_dir <${GLOG_INCLUDE_DIR}> lib <${GLOG_LIBRARY}>")

include_directories(SYSTEM ${OPENSSL_INCLUDE_DIR})

include_directories(SYSTEM ${GFLAGS_INCLUDE_DIR})
include_directories(SYSTEM ${GLOG_INCLUDE_DIR})

include_directories(${CMAKE_SOURCE_DIR})

include_directories(${CMAKE_CURRENT_BINARY_DIR}/reactivestreams/include)
include_directories(${GMOCK_SOURCE_DIR}/googlemock/include)
include_directories(${GMOCK_SOURCE_DIR}/googletest/include)

add_subdirectory(yarpl)

add_library(
  ReactiveSocket
  rsocket/RSocket.cpp
  rsocket/RSocket.h
  rsocket/RSocketServer.h
  rsocket/RSocketServer.cpp
  rsocket/RSocketClient.h
  rsocket/RSocketClient.cpp
  rsocket/RSocketRequester.h
  rsocket/RSocketRequester.cpp
  rsocket/RSocketErrors.h
  rsocket/RSocketParameters.cpp
  rsocket/RSocketParameters.h
  rsocket/ConnectionAcceptor.h
  rsocket/ConnectionFactory.h
  rsocket/transports/tcp/TcpConnectionAcceptor.h
  rsocket/transports/tcp/TcpConnectionAcceptor.cpp
  rsocket/transports/tcp/TcpConnectionFactory.h
  rsocket/transports/tcp/TcpConnectionFactory.cpp
  rsocket/RSocketResponder.cpp
  rsocket/RSocketResponder.h
  rsocket/statemachine/ChannelRequester.cpp
  rsocket/statemachine/ChannelRequester.h
  rsocket/statemachine/ChannelResponder.cpp
  rsocket/statemachine/ChannelResponder.h
  rsocket/statemachine/ConsumerBase.cpp
  rsocket/statemachine/ConsumerBase.h
  rsocket/statemachine/PublisherBase.cpp
  rsocket/statemachine/PublisherBase.h
  rsocket/statemachine/RequestResponseRequester.cpp
  rsocket/statemachine/RequestResponseRequester.h
  rsocket/statemachine/RequestResponseResponder.cpp
  rsocket/statemachine/RequestResponseResponder.h
  rsocket/statemachine/StreamStateMachineBase.cpp
  rsocket/statemachine/StreamStateMachineBase.h
  rsocket/statemachine/StreamRequester.cpp
  rsocket/statemachine/StreamRequester.h
  rsocket/statemachine/StreamResponder.cpp
  rsocket/statemachine/StreamResponder.h
  rsocket/statemachine/StreamsFactory.cpp
  rsocket/statemachine/StreamsFactory.h
  rsocket/statemachine/StreamsWriter.h
  rsocket/statemachine/StreamState.cpp
  rsocket/statemachine/StreamState.h
  rsocket/internal/ClientResumeStatusCallback.h
  rsocket/internal/Common.cpp
  rsocket/internal/Common.h
  rsocket/statemachine/RSocketStateMachine.cpp
  rsocket/statemachine/RSocketStateMachine.h
  rsocket/DuplexConnection.h
  rsocket/internal/FollyKeepaliveTimer.cpp
  rsocket/internal/FollyKeepaliveTimer.h
  rsocket/framing/ErrorCode.cpp
  rsocket/framing/ErrorCode.h
  rsocket/framing/Frame.cpp
  rsocket/framing/Frame.h
  rsocket/framing/FrameFlags.cpp
  rsocket/framing/FrameFlags.h
  rsocket/framing/FrameProcessor.h
  rsocket/framing/FrameSerializer.cpp
  rsocket/framing/FrameSerializer.h
  rsocket/framing/FrameTransport.cpp
  rsocket/framing/FrameTransport.h
  rsocket/framing/FrameType.cpp
  rsocket/framing/FrameType.cpp
  rsocket/framing/FrameType.h
  rsocket/framing/FrameType.h
  rsocket/framing/FramedDuplexConnection.cpp
  rsocket/framing/FramedDuplexConnection.h
  rsocket/framing/FramedReader.cpp
  rsocket/framing/FramedReader.h
  rsocket/framing/FramedWriter.cpp
  rsocket/framing/FramedWriter.h
  rsocket/Payload.cpp
  rsocket/Payload.h
  rsocket/internal/ResumeCache.cpp
  rsocket/internal/ResumeCache.h
  rsocket/internal/SetupResumeAcceptor.cpp
  rsocket/internal/SetupResumeAcceptor.h
  rsocket/RSocketStats.cpp
  rsocket/RSocketStats.h
  rsocket/transports/tcp/TcpDuplexConnection.cpp
  rsocket/transports/tcp/TcpDuplexConnection.h
  rsocket/framing/FrameSerializer_v0.cpp
  rsocket/framing/FrameSerializer_v0.h
  rsocket/framing/FrameSerializer_v0_1.cpp
  rsocket/framing/FrameSerializer_v0_1.h
  rsocket/framing/FrameSerializer_v1_0.cpp
  rsocket/framing/FrameSerializer_v1_0.h
  rsocket/internal/ScheduledSubscription.cpp
  rsocket/internal/ScheduledSubscription.h
  rsocket/internal/ScheduledSubscriber.h
  rsocket/internal/ScheduledRSocketResponder.cpp
  rsocket/internal/ScheduledRSocketResponder.h
  rsocket/internal/ScheduledSingleObserver.h
  rsocket/internal/ScheduledSingleSubscription.cpp
  rsocket/internal/ScheduledSingleSubscription.h
  rsocket/internal/RSocketConnectionManager.cpp
  rsocket/internal/RSocketConnectionManager.h
  rsocket/RSocketSetup.cpp
  rsocket/RSocketSetup.h
  rsocket/ResumeManager.h
  rsocket/ColdResumeHandler.h
  rsocket/Exception.h)

target_include_directories(ReactiveSocket PUBLIC "${PROJECT_SOURCE_DIR}/yarpl/include")
target_include_directories(ReactiveSocket PUBLIC "${PROJECT_SOURCE_DIR}/yarpl/src")

target_link_libraries(ReactiveSocket yarpl ${GFLAGS_LIBRARY} ${GLOG_LIBRARY})

target_compile_options(
  ReactiveSocket
  PRIVATE ${EXTRA_CXX_FLAGS})

enable_testing()

install(TARGETS ReactiveSocket DESTINATION lib)
install(DIRECTORY rsocket DESTINATION include FILES_MATCHING PATTERN "*.h")

# CMake doesn't seem to support "transitive" installing, and I can't access the
# "yarpl" target from this file, so just grab the library file directly.
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/yarpl/libyarpl.a" DESTINATION lib)
install(DIRECTORY yarpl/include/yarpl DESTINATION include FILES_MATCHING PATTERN "*.h")

add_executable(
  tests
  test/MocksTest.cpp
  test/PayloadTest.cpp
  test/RSocketClientServerTest.cpp
  test/RSocketTests.h
  test/RequestChannelTest.cpp
  test/RequestResponseTest.cpp
  test/RequestStreamTest.cpp
  test/Test.cpp
  test/framing/FrameTest.cpp
  test/framing/FrameTransportTest.cpp
  test/handlers/HelloStreamRequestHandler.cpp
  test/handlers/HelloStreamRequestHandler.h
  test/internal/AllowanceSemaphoreTest.cpp
  test/internal/FollyKeepaliveTimerTest.cpp
  test/test_utils/MockDuplexConnection.h
  test/test_utils/MockKeepaliveTimer.h
  test/test_utils/MockRequestHandler.h
  test/test_utils/MockStats.h
  test/test_utils/Mocks.h
  test/transport/DuplexConnectionTest.cpp
  test/transport/DuplexConnectionTest.h
  test/transport/TcpDuplexConnectionTest.cpp)

target_link_libraries(
  tests
  ReactiveSocket
  yarpl
  ${GMOCK_LIBS}
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

target_compile_options(
  tests
  PRIVATE ${TEST_CXX_FLAGS})

add_dependencies(tests gmock ReactiveSocket)

add_test(NAME ReactiveSocketTests COMMAND tests)

########################################
# TCK Drivers
########################################

add_executable(
  tckclient
  tck-test/client.cpp
  tck-test/TestFileParser.cpp
  tck-test/TestFileParser.h
  tck-test/FlowableSubscriber.cpp
  tck-test/FlowableSubscriber.h
  tck-test/SingleSubscriber.cpp
  tck-test/SingleSubscriber.h
  tck-test/TestSuite.cpp
  tck-test/TestSuite.h
  tck-test/TestInterpreter.cpp
  tck-test/TestInterpreter.h
  tck-test/TypedCommands.h
  tck-test/BaseSubscriber.cpp
  tck-test/BaseSubscriber.h)

target_link_libraries(
  tckclient
  ReactiveSocket
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

add_executable(
  tckserver
  tck-test/server.cpp
  tck-test/MarbleProcessor.cpp
  tck-test/MarbleProcessor.h
  test/test_utils/StatsPrinter.cpp
  test/test_utils/StatsPrinter.h)

target_link_libraries(
  tckserver
  ReactiveSocket
  yarpl
  ${GFLAGS_LIBRARY}
  ${GMOCK_LIBS}
  ${GLOG_LIBRARY}
  ${DOUBLE-CONVERSION})

# Download the latest TCK drivers JAR.
set(TCK_DRIVERS_JAR reactivesocket-tck-drivers-0.9-SNAPSHOT.jar)
join(TCK_DRIVERS_URL
  "https://oss.jfrog.org/libs-snapshot/io/reactivesocket/"
  "reactivesocket-tck-drivers/0.9-SNAPSHOT/${TCK_DRIVERS_JAR}")
message(STATUS "Downloading ${TCK_DRIVERS_URL}")
file(DOWNLOAD ${TCK_DRIVERS_URL} ${CMAKE_SOURCE_DIR}/${TCK_DRIVERS_JAR})

########################################
# Examples
########################################

add_library(
  reactivesocket_examples_util
  examples/util/ExampleSubscriber.cpp
  examples/util/ExampleSubscriber.h
)

target_link_libraries(
  reactivesocket_examples_util
  yarpl
  ReactiveSocket
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

# request-response-hello-world

add_executable(
  example_request-response-hello-world-server
  examples/request-response-hello-world/RequestResponseHelloWorld_Server.cpp
)

target_link_libraries(
  example_request-response-hello-world-server
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

add_executable(
  example_request-response-hello-world-client
  examples/request-response-hello-world/RequestResponseHelloWorld_Client.cpp
)

target_link_libraries(
  example_request-response-hello-world-client
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

# fire-and-forget-hello-world

add_executable(
  example_fire-and-forget-hello-world-server
  examples/fire-and-forget-hello-world/FireAndForgetHelloWorld_Server.cpp
)

target_link_libraries(
  example_fire-and-forget-hello-world-server
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

add_executable(
  example_fire-and-forget-hello-world-client
  examples/fire-and-forget-hello-world/FireAndForgetHelloWorld_Client.cpp
)

target_link_libraries(
  example_fire-and-forget-hello-world-client
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})


# stream-hello-world

add_executable(
  example_stream-hello-world-server
  examples/stream-hello-world/StreamHelloWorld_Server.cpp
)

target_link_libraries(
  example_stream-hello-world-server
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

add_executable(
  example_stream-hello-world-client
  examples/stream-hello-world/StreamHelloWorld_Client.cpp
)

target_link_libraries(
  example_stream-hello-world-client
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

# channel-hello-world

add_executable(
  example_channel-hello-world-server
  examples/channel-hello-world/ChannelHelloWorld_Server.cpp
)

target_link_libraries(
  example_channel-hello-world-server
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

add_executable(
  example_channel-hello-world-client
  examples/channel-hello-world/ChannelHelloWorld_Client.cpp
)

target_link_libraries(
  example_channel-hello-world-client
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

# stream-observable-to-flowable

add_executable(
  example_observable-to-flowable-server
  examples/stream-observable-to-flowable/StreamObservableToFlowable_Server.cpp
)

target_link_libraries(
  example_observable-to-flowable-server
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

add_executable(
  example_observable-to-flowable-client
  examples/stream-observable-to-flowable/StreamObservableToFlowable_Client.cpp
)

target_link_libraries(
  example_observable-to-flowable-client
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

# conditional-request-handling

add_executable(
  example_conditional-request-handling-server
  examples/conditional-request-handling/ConditionalRequestHandling_Server.cpp
  examples/conditional-request-handling/TextRequestHandler.h
  examples/conditional-request-handling/TextRequestHandler.cpp
  examples/conditional-request-handling/JsonRequestHandler.cpp
  examples/conditional-request-handling/JsonRequestHandler.h
)

target_link_libraries(
  example_conditional-request-handling-server
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})

add_executable(
  example_conditional-request-handling-client
  examples/conditional-request-handling/ConditionalRequestHandling_Client.cpp
)

target_link_libraries(
  example_conditional-request-handling-client
  ReactiveSocket
  reactivesocket_examples_util
  yarpl
  ${GFLAGS_LIBRARY}
  ${GLOG_LIBRARY})


########################################
# End Examples
########################################

if(BUILD_BENCHMARKS)
  ExternalProject_Add(
      google_benchmark
      URL ${CMAKE_SOURCE_DIR}/benchmark-1.1.0.zip
      URL_MD5 c3c5cca410a1959efc93946f1739547f
      CMAKE_ARGS "-DCMAKE_BUILD_TYPE=Release"
      INSTALL_COMMAND ""
  )

  ExternalProject_Get_Property(google_benchmark source_dir)
  set(GOOGLE_BENCHMARK_SOURCE_DIR ${source_dir})
  ExternalProject_Get_Property(google_benchmark binary_dir)
  set(GOOGLE_BENCHMARK_BINARY_DIR ${binary_dir})

  set(
      GOOGLE_BENCHMARK_LIBS
      ${GOOGLE_BENCHMARK_BINARY_DIR}/src/${CMAKE_STATIC_LIBRARY_PREFIX}benchmark${CMAKE_STATIC_LIBRARY_SUFFIX}
  )

  include_directories(${GOOGLE_BENCHMARK_SOURCE_DIR}/include)
  include_directories(${CMAKE_SOURCE_DIR}/experimental)

  function(benchmark name file)
      add_executable(${name} ${file})
      target_link_libraries(
          ${name}
          ReactiveSocket
          yarpl
          ${GOOGLE_BENCHMARK_LIBS}
          ${GFLAGS_LIBRARY}
          ${GLOG_LIBRARY})
      add_dependencies(
          ${name}
          google_benchmark)
  endfunction()

  add_subdirectory(benchmarks)
  add_subdirectory(yarpl/perf)
endif(BUILD_BENCHMARKS)
