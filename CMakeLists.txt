# * TODO: _VARIADIC_MAX
# * TODO: Windows build: CRT selection
# * TODO: Windows build: multi-configuration
# * TODO: Windows build: library naming
# * TODO: BOOST_THREAD_VERSION

# TODO: Boost link mode

# TODO: Windows build: NOMINMAX, UNICODE, _UNICODE, _CONSOLE, _CRT_SECURE_NO_WARNINGS
# TODO: Windows build: version minimum
# TODO: OS X version minimum
# TODO: Test filtering and arguments
# TODO: libraries (socket, winsock, etc)
# TODO: Wall/Werror
# TODO: warning suppressions
# TODO: no strict aliasing
# TODO: FindThreads?
# TODO: _FILE_OFFSET_BITS
# TODO: FindSasl and SASL
# TODO: Check for support for __thread
# TODO: Check for __declspec(thread)
# TODO: Coverage

cmake_minimum_required (VERSION 2.8 FATAL_ERROR)

project(mongoclient C CXX)

option(MONGOCLIENT_USE_SSL "Use SSL" OFF)
option(MONGOCLIENT_DISABLE_DECLSPEC_THREAD "Prohibit use of __declspec(thread)" OFF)
option(MONGOCLIENT_USE_SASL "Use SASL" OFF)

if (WIN32)
  option(MONGOCLIENT_STATIC_WINDOWS "Link to static windows runtime" OFF)
endif()

set (CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/lib/cmake")

set (MONGOCLIENT_VERSION_MAJOR 0)
set (MONGOCLIENT_VERSION_MINOR 8)
set (MONGOCLIENT_VERSION_PATCH 0)
set (MONGOCLIENT_VERSION_EXTRA -pre)

set (MONGOCLIENT_LIB_VERSION ${MONGOCLIENT_VERSION_MAJOR}.${MONGOCLIENT_VERSION_MINOR}.${MONGOCLIENT_VERSION_PATCH})
set (MONGOCLIENT_VERSION ${MONGOCLIENT_VERSION_MAJOR}.${MONGOCLIENT_VERSION_MINOR}.${MONGOCLIENT_VERSION_PATCH}${MONGOCLIENT_VERSION_EXTRA})

find_package(Git REQUIRED)
execute_process(
  COMMAND ${GIT_EXECUTABLE} rev-parse HEAD
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  OUTPUT_VARIABLE MONGOCLIENT_GIT_REVISION
  OUTPUT_STRIP_TRAILING_WHITESPACE)

enable_testing()

add_subdirectory (src)

include (InstallRequiredSystemLibraries)
set (CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/APACHE-2.0.txt")
set (CPACK_PACKAGE_VERSION_MAJOR "${MONGOCLIENT_VERSION_MAJOR}")
set (CPACK_PACKAGE_VERSION_MINOR "${MONGOCLIENT_VERSION_MINOR}")
set (CPACK_PACKAGE_VERSION_PATCH "${MONGOCLIENT_VERSION_PATCH}")
include (CPack)
