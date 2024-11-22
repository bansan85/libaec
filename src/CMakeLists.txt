# Main library aec
add_library(aec OBJECT
  encode.c
  encode_accessors.c
  decode.c
  vector.c)

target_include_directories(aec
  PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>"
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/../include>"
  "$<INSTALL_INTERFACE:include>")

# Handle visibility of symbols. Compatible with gnulib's gl_VISIBILITY
include(CheckCCompilerFlag)
check_c_compiler_flag(-fvisibility=hidden COMPILER_HAS_HIDDEN_VISIBILITY)

# Create both static and shared aec library.
add_library(aec_static STATIC "$<TARGET_OBJECTS:aec>")
target_link_libraries(aec_static PUBLIC aec)
set_target_properties(aec_static
  PROPERTIES
  OUTPUT_NAME $<IF:$<BOOL:${MSVC}>,aec-static,aec>)
target_compile_definitions(aec_static PRIVATE LIBAEC_BUILD)

add_library(aec_shared SHARED "$<TARGET_OBJECTS:aec>")
target_link_libraries(aec_shared PUBLIC aec)
set_target_properties(aec_shared
  PROPERTIES
  VERSION 0.1.3
  SOVERSION 0
  OUTPUT_NAME aec
  PUBLIC_HEADER ${CMAKE_CURRENT_BINARY_DIR}/../include/libaec.h)
target_compile_definitions(aec_shared PRIVATE LIBAEC_BUILD LIBAEC_SHARED)

# Wrapper for compatibility with szip
add_library(sz OBJECT sz_compat.c)
target_link_libraries(sz PUBLIC aec)

# Create both static and shared szip library.
add_library(sz_static STATIC "$<TARGET_OBJECTS:sz>" "$<TARGET_OBJECTS:aec>")
set_target_properties(sz_static
  PROPERTIES
  OUTPUT_NAME $<IF:$<BOOL:${MSVC}>,szip-static,sz>)
target_link_libraries(sz_static PUBLIC sz)
target_compile_definitions(sz_static PRIVATE LIBAEC_BUILD)

add_library(sz_shared SHARED "$<TARGET_OBJECTS:sz>" "$<TARGET_OBJECTS:aec>")
target_link_libraries(sz_shared PUBLIC sz)
set_target_properties(sz_shared
  PROPERTIES
  VERSION 2.0.1
  SOVERSION 2
  OUTPUT_NAME $<IF:$<BOOL:${MSVC}>,szip,sz>
  PUBLIC_HEADER ../include/szlib.h)
target_compile_definitions(sz_shared PRIVATE LIBAEC_BUILD LIBAEC_SHARED)

if(COMPILER_HAS_HIDDEN_VISIBILITY)
  target_compile_definitions(aec_static PUBLIC HAVE_VISIBILITY)
  target_compile_definitions(aec_shared PUBLIC HAVE_VISIBILITY)
  target_compile_definitions(sz_static PUBLIC HAVE_VISIBILITY)
  target_compile_definitions(sz_shared PUBLIC HAVE_VISIBILITY)
endif()

# Simple executable for testing and benchmarking.
add_executable(graec graec.c)
target_link_libraries(graec aec)

include(GNUInstallDirs)
if(UNIX)
  # The shell scripts for benchmarking are supported on unix only
  add_executable(utime EXCLUDE_FROM_ALL utime.c)
  target_include_directories(utime PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/..")
  add_custom_target(bench
    COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/benc.sh
    ${CMAKE_CURRENT_SOURCE_DIR}/../data/typical.rz
    COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/bdec.sh
    DEPENDS graec utime)
endif()

install(TARGETS aec aec_static aec_shared sz sz_static sz_shared
        EXPORT ${PROJECT_NAME}Targets)

install(
  EXPORT ${PROJECT_NAME}Targets
  FILE ${PROJECT_NAME}-targets.cmake
  NAMESPACE ${PROJECT_NAME}::
  DESTINATION cmake)
