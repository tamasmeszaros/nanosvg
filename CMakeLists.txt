cmake_minimum_required(VERSION 3.10)

project(NanoSVG C)

# Choose to build the library or just install the headers
option(NanoSVG_HEADER_ONLY "If enabled static library will not be built." OFF)

# The nanosvg_headeronly target will always be usable downstream and will not depend on any external binary
add_library(nanosvg_headeronly INTERFACE)

set(SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/src)

target_include_directories(nanosvg_headeronly INTERFACE $<INSTALL_INTERFACE:include>)
target_link_libraries(nanosvg_headeronly INTERFACE m) # link to cmath

# If downstream links to nanosvg_headeronly, the nanosvg.c source file will be added to the target (but only ones -- no symbol duplication)
# nanosvg.c will be available from the include dir next to nanosvg.h 
target_sources(nanosvg_headeronly INTERFACE $<INSTALL_INTERFACE:include/nanosvg/nanosvg.c> $<BUILD_INTERFACE:${SRC_DIR}/nanosvg.c>)

# The default target to link to for downstream
add_library(nanosvg INTERFACE)

add_library(nanosvgrast_headeronly INTERFACE)
target_link_libraries(nanosvgrast_headeronly INTERFACE nanosvg_headeronly)
target_sources(nanosvgrast_headeronly INTERFACE $<INSTALL_INTERFACE:include/nanosvg/nanosvgrast.c> $<BUILD_INTERFACE:${SRC_DIR}/nanosvgrast.c>)

add_library(nanosvgrast INTERFACE)

if(NOT NanoSVG_HEADER_ONLY)
    # When the compiled library is installed, the nanosvg target will link to the compiled binary
    set(_nanosvg_lib nanosvg_lib)
    add_library(${_nanosvg_lib} ${SRC_DIR}/nanosvg.c ${SRC_DIR}/nanosvg.h ${SRC_DIR}/nanosvgrast.h)
    set_target_properties(${_nanosvg_lib} PROPERTIES OUTPUT_NAME "nanosvg")
    set_target_properties(${_nanosvg_lib} PROPERTIES DEBUG_POSTFIX "d")
    set_target_properties(${_nanosvg_lib} PROPERTIES SOVERSION "1.0")
    target_include_directories(${_nanosvg_lib} PUBLIC $<INSTALL_INTERFACE:include>)
    target_link_libraries(${_nanosvg_lib} PRIVATE m)
    target_include_directories(${_nanosvg_lib} PRIVATE ${SRC_DIR})
    
    target_link_libraries(nanosvg INTERFACE ${_nanosvg_lib})

    # Same for nanosvgrast
    set(_nanosvgrast_lib nanosvgrast_lib)
    add_library(${_nanosvgrast_lib} ${SRC_DIR}/nanosvgrast.c ${SRC_DIR}/nanosvgrast.h)
    set_target_properties(${_nanosvgrast_lib} PROPERTIES OUTPUT_NAME "nanosvgrast")
    set_target_properties(${_nanosvgrast_lib} PROPERTIES DEBUG_POSTFIX "d")
    set_target_properties(${_nanosvgrast_lib} PROPERTIES SOVERSION "1.0")
    target_link_libraries(${_nanosvgrast_lib} PUBLIC nanosvg)
    target_include_directories(${_nanosvgrast_lib} PRIVATE ${SRC_DIR})

    target_link_libraries(nanosvgrast INTERFACE ${_nanosvgrast_lib})
else()
    # When only the headers are installed, the nanosvg will link to nanosvg_headeronly 
    target_link_libraries(nanosvg INTERFACE nanosvg_headeronly)
    target_link_libraries(nanosvgrast INTERFACE nanosvgrast_headeronly)
endif()

# Installation and export:

include(CMakePackageConfigHelpers)

write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"     
    VERSION 1.0
    COMPATIBILITY AnyNewerVersion
)

install(TARGETS nanosvg nanosvg_headeronly ${_nanosvg_lib} nanosvgrast nanosvgrast_headeronly ${_nanosvgrast_lib}
        EXPORT ${PROJECT_NAME}Targets
)

export(EXPORT ${PROJECT_NAME}Targets 
       FILE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Targets.cmake" 
       NAMESPACE ${PROJECT_NAME}::
)

set(ConfigPackageLocation lib/cmake/${PROJECT_NAME})

configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
  INSTALL_DESTINATION ${ConfigPackageLocation}
  NO_CHECK_REQUIRED_COMPONENTS_MACRO
)

install(
    FILES
      ${SRC_DIR}/nanosvg.h
      ${SRC_DIR}/nanosvg.c
      ${SRC_DIR}/nanosvgrast.h
      ${SRC_DIR}/nanosvgrast.c
    DESTINATION
      include/nanosvg
  )

install(EXPORT ${PROJECT_NAME}Targets
  FILE
    ${PROJECT_NAME}Targets.cmake
  NAMESPACE
    ${PROJECT_NAME}::
  DESTINATION
    ${ConfigPackageLocation}
)

install(
  FILES
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
  DESTINATION
    ${ConfigPackageLocation}
)