file(GLOB SOURCES CONFIGURE_DEPENDS ./*.cpp)
file(GLOB HEADERS CONFIGURE_DEPENDS ./*.h)

if(EMSCRIPTEN)
  add_library(pmp STATIC ${SOURCES} ${HEADERS})
else()
  add_library(pmp ${SOURCES} ${HEADERS})
  set_property(TARGET pmp PROPERTY WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

target_compile_features(pmp PUBLIC cxx_std_20)

# ── Compile definitions ──────────────────────────────────────────────────────
if(PMP_SCALAR_TYPE EQUAL 64)
  target_compile_definitions(pmp PUBLIC PMP_SCALAR_TYPE_64)
endif()
if(PMP_INDEX_TYPE EQUAL 64)
  target_compile_definitions(pmp PUBLIC PMP_INDEX_TYPE_64)
endif()
if(TUDO)
  target_compile_definitions(pmp PUBLIC TUDO)
endif()

# ── Compiler options ─────────────────────────────────────────────────────────
if(PMP_COMMON_CXX_FLAGS)
  if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
    target_compile_options(pmp PRIVATE ${PMP_COMMON_CXX_FLAGS})
  endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
  target_compile_options(pmp PRIVATE -Wno-deprecated-copy)
  if(CMAKE_CXX_COMPILER_VERSION VERSION_GREATER "7.0.0")
    target_compile_options(pmp PRIVATE -Wno-int-in-bool-context)
  endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang" AND NOT EMSCRIPTEN)
  target_compile_options(pmp PRIVATE --system-header-prefix=Eigen)
endif()

if(WIN32)
  target_compile_definitions(pmp PRIVATE
    _USE_MATH_DEFINES NOMINMAX _CRT_SECURE_NO_WARNINGS)
endif()

if(MSVC)
  target_compile_options(pmp PRIVATE /wd4267 /wd4244 /wd4305)
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Debug" AND NOT WIN32)
  target_compile_options(pmp PRIVATE -ggdb)
endif()

# ── OpenMP ───────────────────────────────────────────────────────────────────
find_package(OpenMP)
if(OpenMP_CXX_FOUND)
  target_link_libraries(pmp PUBLIC OpenMP::OpenMP_CXX)
endif()

set_target_properties(pmp PROPERTIES VERSION ${PROJECT_VERSION})

if(WITH_CLANG_TIDY)
  set_target_properties(pmp PROPERTIES CXX_CLANG_TIDY "${CLANG_TIDY_COMMAND}")
endif()

# ── Include directories ──────────────────────────────────────────────────────
if(NOT EMSCRIPTEN AND PMP_INSTALL)
  target_include_directories(pmp
    PUBLIC
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../>
      $<INSTALL_INTERFACE:include/>)

  install(TARGETS pmp
    EXPORT pmpTargets
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    RUNTIME DESTINATION bin
    INCLUDES DESTINATION include)

  install(FILES ${HEADERS} DESTINATION include/pmp/)
else()
  target_include_directories(pmp
    PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/../)
endif()

target_link_libraries(pmp PUBLIC Eigen3::Eigen)

# ── Viewers ──────────────────────────────────────────────────────────────────
if(PMP_BUILD_VIEWERS)
  add_subdirectory(viewers)
endif()

include(algorithms/CMakeLists.txt)
include(io/CMakeLists.txt)

if(CMAKE_C_COMPILER_ID MATCHES "GNU"
   AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 15)
  set_source_files_properties(
    algorithms/decimation.cpp
    PROPERTIES COMPILE_FLAGS "-Wno-array-bounds -Wno-stringop-overflow")
endif()
