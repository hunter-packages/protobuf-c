cmake_minimum_required(VERSION 3.1 FATAL_ERROR)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
include(HunterGate)
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.20.77.tar.gz"
    SHA1 "3afa578a16b2998e2338d7d6ccfbec9c7027fccb"
)

project(protobuf-c VERSION 1.3.0)

include(CPack)
include(CTest)

option(BUILD_PROTO3 "BUILD_PROTO3" ON)

include(TestBigEndian)
test_big_endian(WORDS_BIGENDIAN)

set(defs
  PACKAGE_STRING="${PROJECT_NAME} ${PROJECT_VERSION}"
  WORDS_BIGENDIAN=${WORDS_BIGENDIAN})

if(MSVC)
  # using Visual Studio C++
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /wd4267 /wd4244")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4267 /wd4244")
endif()

add_library(protobuf-c
  protobuf-c/protobuf-c.c
  protobuf-c/protobuf-c.h)
target_compile_definitions(protobuf-c PRIVATE ${defs})
target_include_directories(protobuf-c PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/protobuf-c>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)

hunter_add_package(Protobuf)
find_package(Protobuf CONFIG REQUIRED)

set(PROTOC_GEN_C_SRC
    protoc-c/c_bytes_field.cc
    protoc-c/c_bytes_field.h
    protoc-c/c_enum.cc
    protoc-c/c_enum_field.cc
    protoc-c/c_enum_field.h
    protoc-c/c_enum.h
    protoc-c/c_extension.cc
    protoc-c/c_extension.h
    protoc-c/c_field.cc
    protoc-c/c_field.h
    protoc-c/c_file.cc
    protoc-c/c_file.h
    protoc-c/c_generator.cc
    protoc-c/c_generator.h
    protoc-c/c_helpers.cc
    protoc-c/c_helpers.h
    protoc-c/c_message.cc
    protoc-c/c_message_field.cc
    protoc-c/c_message_field.h
    protoc-c/c_message.h
    protoc-c/c_primitive_field.cc
    protoc-c/c_primitive_field.h
    protoc-c/c_service.cc
    protoc-c/c_service.h
    protoc-c/c_string_field.cc
    protoc-c/c_string_field.h
    protoc-c/main.cc)
add_executable(protoc-gen-c ${PROTOC_GEN_C_SRC})
target_include_directories(protoc-gen-c PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/protoc-c>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/protobuf-c>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
  $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
target_compile_definitions(protoc-gen-c PRIVATE ${defs})
target_link_libraries(protoc-gen-c PUBLIC
  protobuf::libprotobuf protobuf::libprotoc)

if(BUILD_PROTO3)
  target_compile_definitions(protoc-gen-c PUBLIC HAVE_PROTO3)
endif()

if(CMAKE_HOST_UNIX)
  add_custom_command(
    OUTPUT protoc-c
    COMMAND ${CMAKE_COMMAND} -E create_symlink
            $<TARGET_FILE:protoc-gen-c> protoc-c
    DEPENDS $<TARGET_FILE:protoc-gen-c>
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    VERBATIM)
  add_custom_target(protoc-c-symlink ALL DEPENDS protoc-c)
endif()

function(generate_test_sources PROTO_FILE SRC HDR)
	add_custom_command(
    OUTPUT ${SRC} ${HDR}
    COMMAND ${PROTOBUF_PROTOC_EXECUTABLE}
    ARGS --plugin=$<TARGET_FILE:protoc-gen-c>
         -I${CMAKE_CURRENT_SOURCE_DIR}
         --c_out=${CMAKE_CURRENT_BINARY_DIR}
         ${PROTO_FILE}
    DEPENDS protoc-gen-c)
endfunction()

if(BUILD_TESTING)
  generate_test_sources(t/test.proto t/test.pb-c.c t/test.pb-c.h)

  add_executable(test-generated-code
    t/generated-code/test-generated-code.c
    t/test.pb-c.c t/test.pb-c.h )
  target_link_libraries(test-generated-code PUBLIC protobuf-c)

  add_custom_command(
    OUTPUT t/test-full.pb.cc t/test-full.pb.h
    COMMAND ${PROTOBUF_PROTOC_EXECUTABLE}
    ARGS --cpp_out=${CMAKE_CURRENT_BINARY_DIR}
         -I${CMAKE_CURRENT_SOURCE_DIR}
         t/test-full.proto)

  generate_test_sources(
    t/test-full.proto t/test-full.pb-c.c t/test-full.pb-c.h)

  add_executable(cxx-generate-packed-data
    t/generated-code2/cxx-generate-packed-data.cc
    t/test-full.pb.h t/test-full.pb.cc)
  target_include_directories(cxx-generate-packed-data PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)
  target_link_libraries(cxx-generate-packed-data ${PROTOBUF_LIBRARY})

  file(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/t/generated-code2)
  add_custom_command(OUTPUT t/generated-code2/test-full-cxx-output.inc
    COMMAND ${CMAKE_CURRENT_BINARY_DIR}/cxx-generate-packed-data
            >t/generated-code2/test-full-cxx-output.inc
    DEPENDS cxx-generate-packed-data)

  generate_test_sources(
    t/test-optimized.proto
    t/test-optimized.pb-c.c
    t/test-optimized.pb-c.h)

  add_executable(test-generated-code2
    t/generated-code2/test-generated-code2.c
    t/generated-code2/test-full-cxx-output.inc
    t/test-full.pb-c.h
    t/test-full.pb-c.c
    t/test-optimized.pb-c.h
    t/test-optimized.pb-c.c)
  target_link_libraries(test-generated-code2 PUBLIC protobuf-c)

  generate_test_sources(
    t/issue220/issue220.proto
    t/issue220/issue220.pb-c.c
    t/issue220/issue220.pb-c.h)
  add_executable(
    test-issue220
    t/issue220/issue220.c
    t/issue220/issue220.pb-c.c
    t/issue220/issue220.pb-c.h)
  target_link_libraries(test-issue220 PUBLIC protobuf-c)

  generate_test_sources(
    t/issue251/issue251.proto
    t/issue251/issue251.pb-c.c
    t/issue251/issue251.pb-c.h)
  add_executable(
    test-issue251
    t/issue251/issue251.c
    t/issue251/issue251.pb-c.c
    t/issue251/issue251.pb-c.h)
  target_link_libraries(test-issue251 protobuf-c)

  add_executable(test-version t/version/version.c)
  target_link_libraries(test-version PUBLIC protobuf-c)

  generate_test_sources(
    t/test-proto3.proto
    t/test-proto3.pb-c.c
    t/test-proto3.pb-c.h)
  add_executable(
    test-generated-code3
    t/generated-code/test-generated-code.c
    t/test-proto3.pb-c.c
    t/test-proto3.pb-c.h)
  target_compile_definitions(test-generated-code3 PUBLIC PROTO3)
  target_link_libraries(test-generated-code3 PUBLIC protobuf-c)

  add_test(test-generated-code test-generated-code)
  add_test(test-generated-code2 test-generated-code2)
  add_test(test-generated-code3 test-generated-code3)
  add_test(test-issue220 test-issue220)
  add_test(test-issue251 test-issue251)
  add_test(test-version test-version)
endif()

# Installation (https://github.com/forexample/package-example)

# Layout. This works for all platforms:
#   * <prefix>/lib/cmake/<PROJECT-NAME>
#   * <prefix>/lib/
#   * <prefix>/include/
set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

# Configuration
set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

# Include module with fuction 'write_basic_package_version_file'
include(CMakePackageConfigHelpers)

# Configure '<PROJECT-NAME>ConfigVersion.cmake'
# Use:
#   * PROJECT_VERSION
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Configure '<PROJECT-NAME>Config.cmake'
# Use variables:
#   * TARGETS_EXPORT_NAME
#   * PROJECT_NAME
configure_package_config_file(
    "cmake/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

# Targets:
#   * <prefix>/lib/libprotobuf-c.a
#   * header location after install: <prefix>/include/protobuf-c/protobuf-c.h
#   * headers can be included by C++ code `#include <protobuf-c/protobuf-c.h>`
install(
    TARGETS protobuf-c protoc-gen-c
    EXPORT "${TARGETS_EXPORT_NAME}"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
    RUNTIME DESTINATION "bin"
    INCLUDES DESTINATION "${include_install_dir}"
)

if(CMAKE_HOST_UNIX)
  configure_file(
    "cmake/InstallSymlink.cmake.in"
    "${CMAKE_CURRENT_BINARY_DIR}/cmake/InstallSymlink.cmake"
    @ONLY)
  install(SCRIPT ${CMAKE_CURRENT_BINARY_DIR}/cmake/InstallSymlink.cmake)
endif()

# Headers:
#   * protobuf-c/protobuf-c.h -> <prefix>/include/protobuf-c/protobuf-c.h
install(
    FILES "${CMAKE_CURRENT_SOURCE_DIR}/protobuf-c/protobuf-c.h"
    DESTINATION "${include_install_dir}/protobuf-c"
)

# Config
#   * <prefix>/lib/cmake/protobuf-c/protobuf-cConfig.cmake
#   * <prefix>/lib/cmake/protobuf-c/protobuf-cConfigVersion.cmake
install(
    FILES "${project_config}" "${version_config}"
    DESTINATION "${config_install_dir}"
)

# Config
#   * <prefix>/lib/cmake/protobuf-c/protobuf-cTargets.cmake
install(
    EXPORT "${TARGETS_EXPORT_NAME}"
    NAMESPACE "${namespace}"
    DESTINATION "${config_install_dir}"
)
