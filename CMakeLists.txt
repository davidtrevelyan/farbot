cmake_minimum_required(VERSION 3.10)
project(farbot)

set(CMAKE_CXX_STANDARD 17)
find_package(Threads REQUIRED)

# Enable testing
include(GoogleTest)
enable_testing()
set(GTEST_DIR "${CMAKE_CURRENT_SOURCE_DIR}/googletest/googletest")

# Set a default build type if none was specified
set(default_build_type "Release")
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/.git")
  set(default_build_type "Debug")
endif()

if(NOT CMAKE_BUILD_TYPE)
  message(STATUS "Setting build type to '${default_build_type}' as none was specified.")
  set(CMAKE_BUILD_TYPE "${default_build_type}" CACHE
      STRING "Choose the type of build." FORCE)
  # Set the possible values of build type for cmake-gui
  set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS
    "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

if(CMAKE_BUILD_TYPE MATCHES "Debug")
  add_definitions("-DDEBUG=1" "-D_DEBUG=1")
  if ((("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang") OR ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")) AND (NOT CMAKE_CROSSCOMPILING))
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -fno-omit-frame-pointer -fsanitize=undefined,thread")
    set(CMAKE_LINKER_FLAGS_DEBUG "${CMAKE_LINKER_FLAGS_DEBUG} -fno-omit-frame-pointer -fsanitize=undefined,thread")
    set(SANITIZERS_ENABLED 1)
  else()
    set(SANITIZERS_ENABLED 0)
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
    set(CMAKE_LINKER_FLAGS_DEBUG "${CMAKE_LINKER_FLAGS_DEBUG}")
  endif()
elseif(CMAKE_BUILD_TYPE MATCHES "Release")
  set(SANITIZERS_ENABLED 0)
  add_definitions("-DNDEBUG=1")
else()
  message(FATAL_ERROR "Unknown build configuration \"${CMAKE_BUILD_TYPE}\": must be Debug or Release")
endif()

#increase warning level
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wno-missing-field-initializers -Wshadow-all -Wshorten-64-to-32 -Wstrict-aliasing -Wuninitialized -Wunused-parameter -Wsign-compare -Wint-conversion -Wconditional-uninitialized -Wreorder -Wconstant-conversion -Wunused-private-field -Wbool-conversion -Wextra-semi -Wno-ignored-qualifiers -Wunreachable-code")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wno-missing-field-initializers -Wshadow -Wstrict-aliasing -Wuninitialized -Wunused-parameter -Wsign-compare -Wreorder -Wno-ignored-qualifiers -Wunreachable-code")
  string(APPEND CMAKE_SHARED_LINKER_FLAGS " -Wl,--no-undefined -Wl,-rpath-link,${CMAKE_SYSROOT}/lib")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
endif()

# clang needs libatomic
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
 set(CMAKE_CXX_LINK_FLAGS "${CMAKE_CXX_LINK_FLAGS} -latomic")
endif()

set(FARBOT_LIB_TARGET ${PROJECT_NAME})
add_library(${FARBOT_LIB_TARGET} INTERFACE)
target_include_directories(${FARBOT_LIB_TARGET} INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> $<INSTALL_INTERFACE:include>)

include(CMakePackageConfigHelpers)
write_basic_package_version_file("${PROJECT_BINARY_DIR}/farbot-config-version.cmake" VERSION 0.1 COMPATIBILITY AnyNewerVersion)
install(
    TARGETS farbot
    EXPORT farbotTargets
    LIBRARY DESTINATION lib COMPONENT Runtime
    ARCHIVE DESTINATION lib COMPONENT Development
    RUNTIME DESTINATION bin COMPONENT Runtime
    PUBLIC_HEADER DESTINATION include COMPONENT Development
    BUNDLE DESTINATION bin COMPONENT Runtime)

include(CMakePackageConfigHelpers)
configure_package_config_file("${PROJECT_SOURCE_DIR}/cmake/farbot-config.cmake.in" "${PROJECT_BINARY_DIR}/farbot-config.cmake" INSTALL_DESTINATION lib/cmake/farbot)
install(EXPORT farbotTargets DESTINATION lib/cmake/farbot)
install(FILES "${PROJECT_BINARY_DIR}/farbot-config-version.cmake" "${PROJECT_BINARY_DIR}/farbot-config.cmake" DESTINATION lib/cmake/farbot)
install(DIRECTORY ${PROJECT_SOURCE_DIR}/include/ DESTINATION include)

add_executable(farbot_test test/fifo_test.cpp test/compile_all_headers.cpp)
target_link_libraries(farbot_test PRIVATE ${FARBOT_LIB_TARGET} gtest gtest_main Threads::Threads)
gtest_discover_tests(farbot_test TEST_PREFIX farbot:)
