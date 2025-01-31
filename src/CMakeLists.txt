cmake_minimum_required(VERSION 3.17)
project(couchbase C CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 99)

include(${PROJECT_SOURCE_DIR}/cmake/revisions.cmake OPTIONAL)

set(COUCHBASE_CXX_CLIENT_BUILD_TESTS
    OFF
    CACHE BOOL "" FORCE)
set(COUCHBASE_CXX_CLIENT_BUILD_EXAMPLES
    OFF
    CACHE BOOL "" FORCE)
add_subdirectory(deps/couchbase-cxx-client)

set(COUCHBASE_TXNS_CXX_BUILD_TESTS
    OFF
    CACHE BOOL "" FORCE)
set(COUCHBASE_TXNS_CXX_BUILD_EXAMPLES
    OFF
    CACHE BOOL "" FORCE)
set(COUCHBASE_TXNS_CXX_BUILD_DOC
    OFF
    CACHE BOOL "" FORCE)
set(COUCHBASE_TXNS_CXX_CLIENT_EXTERNAL
    ON
    CACHE BOOL "" FORCE)

string(
  REGEX
  REPLACE "[\r\n]| *-I| */I"
          " "
          PHP_INCLUDE_DIRS
          "${COUCHBASE_PHP_INCLUDES}")
separate_arguments(PHP_INCLUDE_DIRS)
message(STATUS "PHP_INCLUDE_DIRS: ${PHP_INCLUDE_DIRS}")
separate_arguments(COUCHBASE_PHP_CFLAGS)
message(STATUS "COUCHBASE_PHP_CFLAGS: ${COUCHBASE_PHP_CFLAGS}")

find_program(GIT git)
if(NOT EXT_GIT_REVISION)
  if(GIT)
    execute_process(
      COMMAND git rev-parse HEAD
      WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
      OUTPUT_STRIP_TRAILING_WHITESPACE
      OUTPUT_VARIABLE EXT_GIT_REVISION)
  else()
    set(EXT_GIT_REVISION "unknown")
  endif()
endif()
if(NOT COUCHBASE_CXX_CLIENT_GIT_REVISION)
  if(GIT)
    execute_process(
      COMMAND git rev-parse HEAD
      WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}/deps/couchbase-cxx-client
      OUTPUT_STRIP_TRAILING_WHITESPACE
      OUTPUT_VARIABLE COUCHBASE_CXX_CLIENT_GIT_REVISION)
  else()
    set(COUCHBASE_CXX_CLIENT_GIT_REVISION "unknown")
  endif()
endif()
configure_file(${PROJECT_SOURCE_DIR}/cmake/ext_build_version.hxx.in
               ${PROJECT_BINARY_DIR}/generated/ext_build_version.hxx @ONLY)

file(GLOB SOURCE_FILES ${PROJECT_SOURCE_DIR}/wrapper/*.cxx)
add_library(couchbase_php_wrapper SHARED ${SOURCE_FILES})
target_include_directories(couchbase_php_wrapper PRIVATE ${PROJECT_BINARY_DIR}/generated)
target_include_directories(couchbase_php_wrapper PRIVATE ${PHP_INCLUDE_DIRS})
target_link_libraries(couchbase_php_wrapper PRIVATE project_options project_warnings couchbase_cxx_client)
if(APPLE)
  target_link_libraries(couchbase_php_wrapper PRIVATE -Wl,-undefined,dynamic_lookup)
endif()
if(WIN32)
  target_compile_options(couchbase_php_wrapper PRIVATE ${COUCHBASE_PHP_CFLAGS} /bigobj)
  target_compile_definitions(couchbase_php_wrapper PRIVATE NOMINMAX)
  target_link_options(couchbase_php_wrapper PUBLIC "${COUCHBASE_PHP_LIB}")
endif()
if(COUCHBASE_PHP_LIBDIR)
  add_custom_command(
    TARGET couchbase_php_wrapper
    POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy "$<TARGET_FILE:couchbase_php_wrapper>"
            "$<TARGET_LINKER_FILE:couchbase_php_wrapper>" "${COUCHBASE_PHP_LIBDIR}/"
    VERBATIM)
endif()
