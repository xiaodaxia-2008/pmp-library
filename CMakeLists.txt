cmake_minimum_required(VERSION 3.21)
project(
  pmp-library
  VERSION 3.0.0
  DESCRIPTION "The Polygon Mesh Processing Library"
  HOMEPAGE_URL "https://www.pmp-library.org"
  LANGUAGES CXX
)

# ── Options ──────────────────────────────────────────────────────────────────
option(PMP_BUILD_EXAMPLES "Build the PMP examples" OFF)
option(PMP_BUILD_TESTS "Build the PMP test programs" OFF)
option(PMP_BUILD_DOCS "Build the PMP documentation" OFF)
option(PMP_BUILD_VIEWERS "Build the PMP viewers" OFF)
option(PMP_BUILD_POLYGONAL "Build Polygonal" OFF)
option(PMP_INSTALL "Install the PMP library and headers" ON)
option(PMP_STRICT_COMPILATION "Treat compiler warnings as errors" ON)
option(PMP_BUILD_REGRESSIONS "Build the PMP regression test programs" OFF)
option(BUILD_SHARED_LIBS "Build using shared libraries" ON)

list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")

# ── Viewer dependencies (only when enabled) ──────────────────────────────────
if(PMP_BUILD_VIEWERS)
  set(GLFW_SOURCE_DIR_INTERNAL "external/glfw-3.4")
  set(GLAD_SOURCE_DIR "external/glad-2.0.8")
  set(IMGUI_SOURCE_DIR "external/imgui-1.92.6")
  set(STB_IMAGE_SOURCE_DIR "external/stb_image-2.28")
  set(STB_IMAGE_WRITE_SOURCE_DIR "external/stb_image_write-1.16")

  add_subdirectory(${STB_IMAGE_WRITE_SOURCE_DIR})
  add_subdirectory(${STB_IMAGE_SOURCE_DIR})

  set(BUILD_SHARED_LIBS OFF CACHE BOOL "")
  set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL "")
  set(GLFW_BUILD_TESTS OFF CACHE BOOL "")
  set(GLFW_BUILD_DOCS OFF CACHE BOOL "")
  set(GLFW_INSTALL OFF CACHE BOOL "")

  find_program(WAYLAND_SCANNER_EXECUTABLE NAMES wayland-scanner)
  if(NOT WAYLAND_SCANNER_EXECUTABLE)
    set(GLFW_BUILD_WAYLAND False)
  endif()

  if(NOT EMSCRIPTEN)
    add_subdirectory(${GLFW_SOURCE_DIR_INTERNAL})
    include_directories(${GLAD_SOURCE_DIR}/include)
  endif()

  add_subdirectory(${IMGUI_SOURCE_DIR})
endif()

# ── Eigen3 ───────────────────────────────────────────────────────────────────
find_package(Eigen3 REQUIRED)

# ── Coverage (debug only) ────────────────────────────────────────────────────
if(CMAKE_BUILD_TYPE STREQUAL "Debug" AND ENABLE_COVERAGE AND NOT WIN32)
  enable_testing()
  set(CMAKE_CXX_FLAGS "-g -O0 --coverage")
  find_program(LCOV lcov REQUIRED)
  find_program(GENHTML genhtml REQUIRED)
  add_custom_target(
    coverage
    COMMAND ${LCOV} --directory . --capture --exclude '*/external/*'
            --exclude '*/tests/*' --exclude '/usr/*' --exclude '/Applications/*'
            --exclude 'v1/*' --output-file coverage.info --ignore-errors unused
    COMMAND ${GENHTML} --demangle-cpp -o coverage coverage.info
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR})
endif()

# ── Default build type ───────────────────────────────────────────────────────
if(NOT CMAKE_BUILD_TYPE)
  message(STATUS "No build type specified. Using 'Release' as default.")
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING
    "The build type. Possible values: Debug Release RelWithDebInfo MinSizeRel."
    FORCE)
endif()

# ── Common compiler flags (applied to pmp target in its CMakeLists.txt) ──────
set(PMP_COMMON_CXX_FLAGS -pedantic -Wall -Wextra -Wshadow)
if(PMP_STRICT_COMPILATION)
  list(APPEND PMP_COMMON_CXX_FLAGS -Werror)
endif()

# ── Scalar / index type (applied to pmp target in its CMakeLists.txt) ────────
if(PMP_SCALAR_TYPE EQUAL 64)
  message(STATUS "Using 64-bit scalar type")
endif()
if(PMP_INDEX_TYPE EQUAL 64)
  message(STATUS "Using 64-bit index type")
endif()

# ── Emscripten global settings ───────────────────────────────────────────────
if(EMSCRIPTEN)
  add_compile_options(-fexceptions)
  add_link_options(-fexceptions)
  add_link_options(
    "SHELL:-sWASM=1 -sUSE_WEBGL2=1 -sUSE_GLFW=3 -sALLOW_MEMORY_GROWTH=1 -sALLOW_TABLE_GROWTH=1 -sSTACK_SIZE=5MB -sEXPORTED_RUNTIME_METHODS=[ccall]"
  )
  set(CMAKE_EXECUTABLE_SUFFIX ".html")
endif()

# ── clang-tidy ───────────────────────────────────────────────────────────────
option(WITH_CLANG_TIDY "Run clang-tidy checks" OFF)
include(clang-tidy)

# ── Subdirectories ───────────────────────────────────────────────────────────
add_subdirectory(src/pmp)

if(PMP_BUILD_POLYGONAL)
  add_subdirectory(src/polygonal)
endif()

if(NOT EMSCRIPTEN)
  if(PMP_BUILD_DOCS)
    add_subdirectory(docs)
  endif()
  if(PMP_BUILD_EXAMPLES)
    add_subdirectory(examples)
  endif()
  if(PMP_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
  endif()
else()
  if(PMP_BUILD_EXAMPLES)
    add_subdirectory(examples)
  endif()
endif()

# ── Install (package config) ─────────────────────────────────────────────────
if(NOT EMSCRIPTEN AND PMP_INSTALL)
  include(CMakePackageConfigHelpers)
  set(PMP_CONFIG_PATH "lib${LIB_SUFFIX}/cmake/pmp")

  configure_package_config_file(
    cmake/pmpConfig.cmake.in pmpConfig.cmake
    INSTALL_DESTINATION "${PMP_CONFIG_PATH}"
    NO_CHECK_REQUIRED_COMPONENTS_MACRO)

  install(EXPORT pmpTargets
    FILE pmpTargets.cmake
    EXPORT_LINK_INTERFACE_LIBRARIES
    DESTINATION "${PMP_CONFIG_PATH}")

  export(TARGETS pmp FILE pmpTargets.cmake)

  write_basic_package_version_file(
    pmpConfigVersion.cmake
    VERSION ${PMP_VERSION}
    COMPATIBILITY SameMajorVersion)

  install(FILES "${PROJECT_BINARY_DIR}/pmpConfig.cmake"
                "${PROJECT_BINARY_DIR}/pmpConfigVersion.cmake"
          DESTINATION "${PMP_CONFIG_PATH}")
endif()

# ── Uninstall target ─────────────────────────────────────────────────────────
if(NOT TARGET uninstall AND PMP_INSTALL)
  configure_file("${CMAKE_MODULE_PATH}/cmake_uninstall.cmake.in"
                 cmake_uninstall.cmake IMMEDIATE @ONLY)
  add_custom_target(uninstall
    "${CMAKE_COMMAND}" -P "${PROJECT_BINARY_DIR}/cmake_uninstall.cmake")
endif()

# ── clang-format ─────────────────────────────────────────────────────────────
include(clang-format)
