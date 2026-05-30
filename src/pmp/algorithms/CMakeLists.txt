# make sure to install headers, use relative path from top-level since we're
# included from above
file(GLOB HEADERS CONFIGURE_DEPENDS ./algorithms/*.h)
file(GLOB SOURCES CONFIGURE_DEPENDS ./algorithms/*.cpp)

if(PMP_INSTALL)
  install(FILES ${HEADERS} DESTINATION include/pmp/algorithms)
endif()

target_sources(pmp PRIVATE "${SOURCES}" "${HEADERS}")
