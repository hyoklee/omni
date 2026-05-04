cmake_minimum_required(VERSION 3.22)
project(omni VERSION 1.0.0)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

set(CMAKE_CXX_STANDARD 17)

find_package(yaml-cpp REQUIRED)

include_directories(
  ${CMAKE_CURRENT_SOURCE_DIR}
  ${YAML_CPP_INCLUDE_DIR}
)

option(AWS "Enable AWS SDK support" OFF)
if(AWS)
  find_package(AWSSDK REQUIRED COMPONENTS core s3)
  add_definitions(-DUSE_AWS)
  include_directories(${AWSSDK_INCLUDE_DIR})
endif()

add_definitions(-DUSE_POCO)
if(${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD")
  message(STATUS "OpenBSD detected: Using manual Poco path resolution")
  include_directories(/usr/local/include)
  foreach(_comp Crypto Foundation Net NetSSL)
    file(GLOB _lib "/usr/local/lib/libPoco${_comp}.so*")
    list(GET _lib 0 _lib_path)
    if(NOT _lib_path)
      message(FATAL_ERROR "Poco ${_comp} library not found in /usr/local/lib")
    endif()
    set(POCO_${_comp}_LIB "${_lib_path}")
  endforeach()
elseif(${CMAKE_SYSTEM_NAME} MATCHES "NetBSD")
  message(STATUS "NetBSD detected: Using manual Poco path resolution")
  include_directories(/usr/pkg/include)
  foreach(_comp Crypto Foundation Net NetSSL)
    file(GLOB _lib "/usr/pkg/lib/libPoco${_comp}.so*")
    list(GET _lib 0 _lib_path)
    if(NOT _lib_path)
      message(FATAL_ERROR "Poco ${_comp} library not found in /usr/pkg/lib")
    endif()
    set(POCO_${_comp}_LIB "${_lib_path}")
  endforeach()
else()
  find_package(Poco REQUIRED COMPONENTS Crypto Foundation Net NetSSL)
endif()

option(HERMES "Enable HERMES support" OFF)
if(HERMES)
  add_definitions(-DUSE_HERMES)
  find_package(Hermes REQUIRED)
  find_package(cereal REQUIRED)
  set(CDIR ${cereal_DIR}/../../../include)
  find_package(Thallium REQUIRED)
  set(TDIR ${Thallium_DIR}/../../../include)
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(Margo REQUIRED margo)
  include_directories(
    ${HERMES_SRC_DIR} ${HERMES_ADAPTER_DIR} ${HERMES_IO_CLIENT_DIR}
    ${TDIR} ${Margo_INCLUDE_DIRS} ${CDIR}
  )
endif()

option(USE_HDF5 "Enable HDF5 support" OFF)
if(USE_HDF5)
  find_package(HDF5 REQUIRED)
  add_definitions(-DUSE_HDF5)
  include_directories(${HDF5_INCLUDE_DIRS})
endif()

option(USE_MPI "Enable MPI support" OFF)
if(USE_MPI)
  find_package(MPI REQUIRED)
  add_definitions(-DUSE_MPI)
endif()

set(OMNI_SOURCES
  OMNI.cc
  format/format_factory.cc
  format/dataset_config.cc
  format/globus_utils.cc
  repo/repo_factory.cc
)

if(USE_HDF5)
  list(APPEND OMNI_SOURCES format/hdf5_dataset_client.cc)
endif()

add_library(omni ${OMNI_SOURCES})

target_include_directories(omni
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/omni>
)

if(TARGET yaml-cpp::yaml-cpp)
  target_link_libraries(omni PUBLIC yaml-cpp::yaml-cpp)
else()
  target_link_libraries(omni PUBLIC yaml-cpp)
endif()

find_library(FILESYSTEM_LIB stdc++fs
  PATHS ${CMAKE_CXX_IMPLICIT_LINK_DIRECTORIES}
)
if(FILESYSTEM_LIB)
  target_link_libraries(omni PUBLIC ${FILESYSTEM_LIB})
endif()

if(HERMES)
  target_link_libraries(omni PUBLIC hermes::hermes_core_client)
endif()

if(AWS)
  target_link_libraries(omni PUBLIC ${AWSSDK_LINK_LIBRARIES})
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "OpenBSD|NetBSD")
  target_link_libraries(omni PUBLIC
    "${POCO_Crypto_LIB}" "${POCO_Foundation_LIB}"
    "${POCO_Net_LIB}" "${POCO_NetSSL_LIB}"
    ssl crypto)
else()
  target_link_libraries(omni PUBLIC Poco::Crypto Poco::Foundation Poco::Net Poco::NetSSL)
endif()

if(USE_HDF5)
  target_link_libraries(omni PUBLIC ${HDF5_LIBRARIES})
endif()

if(USE_MPI)
  target_link_libraries(omni PUBLIC MPI::MPI_CXX)
endif()

# Install library and headers
install(TARGETS omni
  EXPORT omniTargets
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

install(FILES OMNI.h omni_job_config.h omni_processing.h
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/omni
)

install(DIRECTORY format/ repo/
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/omni
  FILES_MATCHING PATTERN "*.h"
)

install(EXPORT omniTargets
  FILE omniTargets.cmake
  NAMESPACE omni::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/omni
)

configure_package_config_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/omniConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/omniConfig.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/omni
)

write_basic_package_version_file(
  omniConfigVersion.cmake
  VERSION ${PROJECT_VERSION}
  COMPATIBILITY AnyNewerVersion
)

install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/omniConfig.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/omniConfigVersion.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/omni
)
