if (MSVC)
  add_definitions("_VARADIC_MAX=10")
endif()

add_definitions("BOOST_THREAD_VERSION=2")

add_subdirectory(third_party)
add_subdirectory(mongo)
