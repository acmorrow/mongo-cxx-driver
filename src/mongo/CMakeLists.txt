add_definitions(-DMONGOCLIENT_CMAKE_BUILD)

if (MONGOCLIENT_USE_SSL)
   find_package (OpenSSL REQUIRED)
endif()
include_directories (${OPENSSL_INCLUDE_DIR})

find_package (Boost COMPONENTS thread system REQUIRED)
include_directories (${Boost_INCLUDE_DIR})
link_directories (${Boost_LIB_DIR})

if (WIN32)
  set (MONGOCLIENT_STATIC_LIB_NAME libmongoclient)
  set (MONGOCLIENT_DYNAMIC_LIB_NAME mongoclient)
else()
  set (MONGOCLIENT_STATIC_LIB_NAME mongoclient)
  set (MONGOCLIENT_DYNAMIC_LIB_NAME mongoclient)
endif()

# On windows, if we are asked to not use the dynamic runtime, we need to reconfigure
# cmakes default choice of /MD to /MT, respecting debug modes, across all configurations.
if (WIN32)
  if (not MONGOCLIENT_DYNAMIC_WINDOWS)
    if (MSVC)
      foreach(flag_var
          CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE
          CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO)
        if (${flag_var} MATCHES "/MD")
          string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
        endif()
      endforeach()
    endif()
  else()
    error("Don't know how to enable dynamic windows for non-MSVC windows compilers")
  endif()
endif()

set (MONGOCLIENT_ABI_ADORNMENT "")
if (WIN32)
  if (DEBUG)
    if (not MONGOCLIENT_DYNAMIC_WINDOWS)
      set (MONGOCLIENT_ABI_ADORNMENT "-sgd")
    else()
      set (MONGOCLIENT_ABI_ADORNMENT "-gd")
    endif()
  else()
    if (not MONGOCLIENT_DYNAMIC_WINDOWS)
      set (MONGOCLIENT_ABI_ADORNMENT "-s")
    endif()
  endif()
endif()

set(MONGOCLIENT_STATIC_LIB_NAME "${MONGOCLIENT_STATIC_LIB_NAME}${MONGOCLIENT_ABI_ADORNMENT}")
set(MONGOCLIENT_DYNAMIC_LIB_NAME "${MONGOCLIENT_DYNAMIC_LIB_NAME}${MONGOCLIENT_ABI_ADORNMENT}")

# We use threads, so ban attempts to specify the single threaded boost library.
if (NOT ${Boost_USE_MULTITHREADED})
  error("Mongoclient requires multithreaded boost libraries")
endif()

# We don't use boost auto linking facility, relying instead on FindBoost getting us something
# in accordance with the various Boost_XXX settings.
add_definitions( -DBOOST_ALL_NO_LIB )

# If we are not using the static libraries, add the proper define to use the dynamic libs. This
# is perhaps not needed since we are not auto linking?
if (not ${Boost_USE_STATIC_LIBS} )
  add_definitions( -DBOOST_ALL_DYN_LIB )
endif()

include(CheckIncludeFiles)
include(CheckSymbolExists)
include(GenerateExportHeader)

check_include_files(execinfo.h MONGOCLIENT_HAVE_EXECINFO_BACKTRACE)
check_symbol_exists(backtrace execinfo.h MONGOCLIENT_HAVE_EXECINFO_BACKTRACE)
check_symbol_exists(backtrace_symbols execinfo.h MONGOCLIENT_HAVE_EXECINFO_BACKTRACE)
check_symbol_exists(backtrace_symbols_fd execinfo.h MONGOCLIENT_HAVE_EXECINFO_BACKTRACE)

check_include_files(unistd.h MONGOCLIENT_HAVE_HEADER_UNISTD_H)

add_subdirectory(base)

add_custom_command (
  OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/base/error_codes.h ${CMAKE_CURRENT_BINARY_DIR}/base/error_codes.cpp
  COMMAND python ${CMAKE_CURRENT_SOURCE_DIR}/base/generate_error_codes.py ${CMAKE_CURRENT_SOURCE_DIR}/base/error_codes.err ${CMAKE_CURRENT_BINARY_DIR}/base/error_codes.h ${CMAKE_CURRENT_BINARY_DIR}/base/error_codes.cpp
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/base/generate_error_codes.py
  MAIN_DEPENDENCY ${CMAKE_CURRENT_SOURCE_DIR}/base/error_codes.err)

# TODO: Someday we should use better names, but for now
# respect the names used in the server codebase
set(MONGO_SSL ${MONGOCLIENT_USE_SSL})

configure_file ("config.h.in" "config.h")
configure_file ("version.h.in" "version.h")

include_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}/..
  ${CMAKE_CURRENT_BINARY_DIR}/..
)

set (libmongoclient_sources
  base/error_codes.cpp
  base/global_initializer.cpp
  base/global_initializer_registerer.cpp
  base/init.cpp
  base/initializer.cpp
  base/initializer_context.cpp
  base/initializer_dependency_graph.cpp
  base/make_string_vector.cpp
  base/parse_number.cpp
  base/status.cpp
  base/string_data.cpp
  bson/bson_validate.cpp
  bson/oid.cpp
  bson/optime.cpp
  bson/util/bson_extract.cpp
  client/connpool.cpp
  client/dbclient.cpp
  client/dbclient_rs.cpp
  client/replica_set_monitor.cpp
  client/dbclientcursor.cpp
  client/gridfs.cpp
  client/init.cpp
  client/sasl_client_authenticate.cpp
  db/jsobj.cpp
  db/json.cpp
  db/dbmessage.cpp
  db/server_options.cpp
  logger/log_manager.cpp
  logger/log_severity.cpp
  logger/logger.cpp
  logger/logstream_builder.cpp
  logger/message_event_utf8_encoder.cpp
  logger/message_log_domain.cpp
  platform/process_id.cpp
  platform/random.cpp
  util/assert_util.cpp
  util/background.cpp
  util/base64.cpp
  util/concurrency/synchronization.cpp
  util/concurrency/thread_name.cpp
  util/fail_point.cpp
  util/fail_point_registry.cpp
  util/fail_point_service.cpp
  util/hex.cpp
  util/log.cpp
  util/md5.cpp
  util/password_digest.cpp
  util/net/httpclient.cpp
  util/net/message.cpp
  util/net/message_port.cpp
  util/net/sock.cpp
  util/net/socket_poll.cpp
  util/net/ssl_manager.cpp
  util/stringutils.cpp
  util/text.cpp
  util/time_support.cpp
  util/timer.cpp
  util/util.cpp
  ../third_party/murmurhash3/MurmurHash3.cpp
)

# TODO SASL

set (libmongoclient_headers

  # List generated headers alphabetically here

  ${CMAKE_CURRENT_BINARY_DIR}/base/error_codes.h
  ${CMAKE_CURRENT_BINARY_DIR}/config.h
  ${CMAKE_CURRENT_BINARY_DIR}/export.h
  ${CMAKE_CURRENT_BINARY_DIR}/version.h

  base/disallow_copying.h
  base/parse_number.h
  base/status-inl.h
  base/status.h
  base/status_with.h
  base/string_data-inl.h
  base/string_data.h
  bson/bson-inl.h
  bson/bson.h
  bson/bson_db.h
  bson/bson_field.h
  bson/bsonelement.h
  bson/bsonmisc.h
  bson/bsonobj.h
  bson/bsonobjbuilder.h
  bson/bsonobjiterator.h
  bson/bsontypes.h
  bson/inline_decls.h
  bson/oid.h
  bson/optime.h
  bson/ordering.h
  bson/util/atomic_int.h
  bson/util/builder.h
  bson/util/misc.h
  client/connpool.h
  client/dbclient.h
  client/dbclient_rs.h
  client/dbclientcursor.h
  client/dbclientinterface.h
  client/export_macros.h
  client/gridfs.h
  client/init.h
  client/redef_macros.h
  client/sasl_client_authenticate.h
  client/undef_macros.h
  db/jsobj.h
  db/json.h
  db/server_options.h
  logger/appender.h
  logger/labeled_level.h
  logger/log_domain.h
  logger/log_manager.h
  logger/log_severity-inl.h
  logger/log_severity.h
  logger/logger.h
  logger/logstream_builder.h
  logger/message_event.h
  logger/message_log_domain.h
  logger/tee.h
  platform/atomic_intrinsics.h
  platform/atomic_intrinsics_gcc_generic.h
  platform/atomic_intrinsics_gcc_intel.h
  platform/atomic_intrinsics_win32.h
  platform/atomic_word.h
  platform/basic.h
  platform/compiler.h
  platform/compiler_gcc.h
  platform/compiler_msvc.h
  platform/cstdint.h
  platform/float_utils.h
  platform/hash_namespace.h
  platform/process_id.h
  platform/unordered_map.h
  platform/windows_basic.h
  util/assert_util.h
  util/background.h
  util/bufreader.h
  util/concurrency/mutex.h
  util/concurrency/thread_name.h
  util/debug_util.h
  util/goodies.h
  util/hex.h
  util/log.h
  util/mongoutils/str.h
  util/net/hostandport.h
  util/net/message.h
  util/net/message_port.h
  util/net/sock.h
  util/net/ssl_manager.h
  util/net/ssl_options.h
  util/time_support.h
)

install (FILES ${libmongoclient_headers} DESTINATION include/mongo COMPONENT dev)

function(add_target_definition target define)
  set_property(TARGET ${target} APPEND PROPERTY COMPILE_DEFINITIONS ${define})
endfunction()

function(add_mongoclient_library_defines library)
  add_target_definition(${library} MONGO_EXPOSE_MACROS)
  if (${MONGOCLIENT_USE_SASL})
    add_target_definition(${library} MONGO_SASL)
  endif()
  if (${MONGOCLIENT_HAVE_HEADER_UNISTD_H})
    add_target_definition(${library} MONGO_HAVE_HEADER_UNISTD_H)
  endif()
  if (${MONGOCLIENT_DISABLE_DECLSPEC_THREAD})
    add_target_definition(${library} MONGO_HAVE___DECLSPEC_THREAD)
  endif()
endfunction()

add_library (libmongoclient_static STATIC ${libmongoclient_sources})
set_target_properties(libmongoclient_static PROPERTIES OUTPUT_NAME ${MONGOCLIENT_STATIC_LIB_NAME})
add_target_definition(libmongoclient_static STATIC_LIBMONGOCLIENT)
add_mongoclient_library_defines(libmongoclient_static)
install (TARGETS libmongoclient_static DESTINATION lib COMPONENT runtime)

# Add gtest include dir
ExternalProject_Get_Property(googletest source_dir)
include_directories(${source_dir}/include)

# Add gtest library dir
ExternalProject_Get_Property(googletest binary_dir)
link_directories(${binary_dir})

add_library (mocklib STATIC
  dbtests/mock/mock_conn_registry.cpp
  dbtests/mock/mock_dbclient_connection.cpp
  dbtests/mock/mock_dbclient_cursor.cpp
  dbtests/mock/mock_remote_db_server.cpp
  dbtests/mock/mock_replica_set.cpp
)
set_target_properties(mocklib PROPERTIES COMPILE_DEFINITIONS "MONGO_EXPOSE_MACROS")


add_library (client_test_main STATIC
  unittest/client_test_main.cpp
)
target_link_libraries (client_test_main libmongoclient_static gtest)
add_dependencies (client_test_main googletest)

add_library (integration_test_main STATIC
  unittest/integration_test_main.cpp
)
add_target_definition(integration_test_main MONGO_EXPOSE_MACROS)

target_link_libraries (integration_test_main libmongoclient_static gtest)
add_dependencies (integration_test_main googletest)

set (unit_tests
    base/parse_number_test
    bson/bson_field_test
    bson/bson_obj_test
    bson/bson_validate_test
    bson/bsonobjbuilder_test
    bson/util/bson_extract_test
    client/dbclient_rs_test
    client/scoped_db_conn_test
    client/replica_set_monitor_test
    dbtests/jsobjtests
    dbtests/jsontests
    dbtests/mock_dbclient_conn_test
    dbtests/mock_replica_set_test
#    dbtests/replica_set_monitor_test
    logger/log_test
    platform/atomic_word_test
    platform/process_id_test
    platform/random_test
    util/net/sock_test
    util/stringutils_test
    util/time_support_test
)

foreach(unittest ${unit_tests})
  get_filename_component(unittest_target ${unittest} NAME)
  add_executable(${unittest_target} ${unittest}.cpp)
  add_target_definition(${unittest_target} MONGO_EXPOSE_MACROS)
  add_target_definition(${unittest_target} STATIC_LIBMONGOCLIENT)
  set_target_properties(${unittest_target} PROPERTIES RUNTIME_OUTPUT_DIRECTORY .)
  target_link_libraries(${unittest_target} libmongoclient_static mocklib client_test_main)
  target_link_libraries(${unittest_target} ${Boost_SYSTEM_LIBRARY} ${Boost_THREAD_LIBRARY})
  if (MONGOCLIENT_USE_SSL)
    target_link_libraries (${unittest_target}
      ${OPENSSL_LIBRARIES}
    )
  endif()
  add_test(${unittest} ${unittest_target})
endforeach()

set (integration_tests
    unittest/dbclient_test
    unittest/gridfs_test
    unittest/sasl_test
)

foreach(integrationtest ${integration_tests})
  get_filename_component(integrationtest_target ${integrationtest} NAME)
  add_executable(${integrationtest_target} ${integrationtest}.cpp)
  add_target_definition(${integrationtest_target} MONGO_EXPOSE_MACROS)
  add_target_definition(${integrationtest_target} STATIC_LIBMONGOCLIENT)
  set_target_properties(${integrationtest_target} PROPERTIES RUNTIME_OUTPUT_DIRECTORY .)
  target_link_libraries(${integrationtest_target} libmongoclient_static mocklib integration_test_main)
  target_link_libraries(${integrationtest_target} ${Boost_SYSTEM_LIBRARY} ${Boost_THREAD_LIBRARY})
  if (MONGOCLIENT_USE_SSL)
    target_link_libraries (${integrationtest_target}
      ${OPENSSL_LIBRARIES}
    )
  endif()
  add_test(${integrationtest} ${integrationtest_target})
endforeach()

if (BUILD_SHARED_LIBS)

  # Do the shared libary later so that we don't mess up flags
  add_library (libmongoclient SHARED ${libmongoclient_sources})
  add_mongoclient_library_defines(libmongoclient)

  set_target_properties(libmongoclient PROPERTIES OUTPUT_NAME ${MONGOCLIENT_DYNAMIC_LIB_NAME})
  set_target_properties(libmongoclient PROPERTIES DEFINE_SYMBOL LIBMONGOCLIENT_EXPORTS)
  add_compiler_export_flags(LIBMONGOCLIENT_EXPORT_FLAGS)
  set_target_properties(libmongoclient PROPERTIES COMPILE_FLAGS ${LIBMONGOCLIENT_EXPORT_FLAGS})
  set_target_properties(libmongoclient PROPERTIES LINK_FLAGS ${LIBMONGOCLIENT_EXPORT_FLAGS})
  generate_export_header(libmongoclient
    EXPORT_FILE_NAME export.h
    EXPORT_MACRO_NAME MONGO_CLIENT_API
    STATIC_DEFINE STATIC_LIBMONGOCLIENT
  )

  set_target_properties( libmongoclient PROPERTIES
    VERSION ${MONGOCLIENT_VERSION}
    SOVERSION ${MONGOCLIENT_VERSION_MAJOR}
  )

  target_link_libraries (libmongoclient
    ${Boost_SYSTEM_LIBRARY}
    ${Boost_THREAD_LIBRARY}
  )

  if (MONGOCLIENT_USE_SSL)
    target_link_libraries (libmongoclient ${OPENSSL_LIBRARIES})
  endif()

  set_target_properties (libmongoclient PROPERTIES
    OUTPUT_NAME mongoclient
    DEFINE_SYMBOL LIBMONGOCLIENT_EXPORTS
    VERSION "${libmongoclient_version_major}.${libmongoclient_version_minor}.${libmongoclient_version_patch}")

  install (TARGETS libmongoclient DESTINATION lib COMPONENT runtime)

  # TODO: macro this
  # TODO: Run these as tests
  add_executable(bsondemo bson/bsondemo/bsondemo.cpp)
  target_link_libraries(bsondemo libmongoclient)

  add_executable(arrayExample client/examples/arrayExample.cpp)
  target_link_libraries(arrayExample libmongoclient)

  add_executable(authTest client/examples/authTest.cpp)
  target_link_libraries(authTest libmongoclient)

  add_executable(clientTest client/examples/clientTest.cpp)
  target_link_libraries(clientTest libmongoclient)

  add_executable(firstExample client/examples/first.cpp)
  target_link_libraries(firstExample libmongoclient)

  add_executable(httpClientTest client/examples/httpClientTest.cpp)
  target_link_libraries(httpClientTest libmongoclient)

  add_executable(insertDemo client/examples/insert_demo.cpp)
  target_link_libraries(insertDemo libmongoclient)

  add_executable(rsExample client/examples/rs.cpp)
  target_link_libraries(rsExample libmongoclient)

  add_executable(secondExample client/examples/second.cpp)
  target_link_libraries(secondExample libmongoclient)

  add_executable(simpleClientDemo client/examples/simple_client_demo.cpp)
  target_link_libraries(simpleClientDemo libmongoclient)

  add_executable(tutorial client/examples/tutorial.cpp)
  target_link_libraries(tutorial libmongoclient)

  add_executable(whereExample client/examples/whereExample.cpp)
  target_link_libraries(whereExample libmongoclient)

endif()

