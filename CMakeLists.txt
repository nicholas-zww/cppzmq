cmake_minimum_required(VERSION 3.11)

list (APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

include (DetectCPPZMQVersion)

project(cppzmq VERSION ${DETECTED_CPPZMQ_VERSION})

if (NOT TARGET libzmq AND NOT TARGET libzmq-static)
  include(FetchContent)

  find_package(ZeroMQ QUIET)

  if (NOT cppzmq_FOUND)
    message(STATUS "libzmq not found. Fetching libzmq...")
    
    FetchContent_Declare(
        libzmq
        GIT_REPOSITORY https://github.com/zeromq/libzmq.git
        GIT_TAG        v4.3.5
    )
    
    # FetchContent_MakeAvailable(libzmq)
    FetchContent_GetProperties(libzmq)
    if (NOT libzmq_POPULATED)
      FetchContent_Populate(libzmq)

      # Check if the patch can be applied
      execute_process(
          COMMAND git apply --check ${CMAKE_CURRENT_SOURCE_DIR}/patches/libzmq.patch
          WORKING_DIRECTORY ${libzmq_SOURCE_DIR}
          RESULT_VARIABLE patch_check_result
          OUTPUT_QUIET
          ERROR_QUIET
      )

      if (patch_check_result EQUAL 0)
          message(STATUS "Applying patch to libzmq...")
          execute_process(
              COMMAND git apply ${CMAKE_CURRENT_SOURCE_DIR}/patches/libzmq.patch
              WORKING_DIRECTORY ${libzmq_SOURCE_DIR}
              RESULT_VARIABLE patch_apply_result
          )

          if (NOT patch_apply_result EQUAL 0)
              message(FATAL_ERROR "Failed to apply patch to libzmq.")
          endif()
      else()
          message(STATUS "Patch already applied or not needed.")
      endif()
    else()
      message(STATUS "libzmq already populated, skipping fetch and patch.")
    endif()

    if (NOT TARGET libzmq)
      add_subdirectory(${libzmq_SOURCE_DIR} ${libzmq_BINARY_DIR})
    endif()
    
    set(CPPZMQ_INCLUDE_DIRS ${libzmq_SOURCE_DIR}/include ${libzmq_BINARY_DIR})
    set(CPPZMQ_LIBRARIES ${libzmq_BINARY_DIR}/libzmq.a)
  else()
    message(STATUS "libzmq found.")
  endif()
endif()

message(STATUS "cppzmq v${cppzmq_VERSION}")

set(CPPZMQ_HEADERS
    zmq.hpp
    zmq_addon.hpp
)

foreach (target cppzmq cppzmq-static)
  add_library(${target} INTERFACE)
  target_include_directories(${target} INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
                                                 $<INSTALL_INTERFACE:include>)
endforeach()

target_link_libraries(cppzmq INTERFACE libzmq)
target_link_libraries(cppzmq-static INTERFACE libzmq-static)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

install(TARGETS cppzmq cppzmq-static
        EXPORT ${PROJECT_NAME}-targets)

install(FILES ${CPPZMQ_HEADERS}
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# GNUInstallDirs "DATADIR" wrong here; CMake search path wants "share".
set(CPPZMQ_CMAKECONFIG_INSTALL_DIR "share/cmake/${PROJECT_NAME}" CACHE STRING "install path for cppzmqConfig.cmake")

configure_file(libzmq-pkg-config/FindZeroMQ.cmake
               libzmq-pkg-config/FindZeroMQ.cmake
               COPYONLY)

export(EXPORT ${PROJECT_NAME}-targets
     FILE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Targets.cmake")
configure_package_config_file(${PROJECT_NAME}Config.cmake.in
                              "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
                              INSTALL_DESTINATION ${CPPZMQ_CMAKECONFIG_INSTALL_DIR})
# Workaround until ARCH_INDEPENDENT flag can be used with cmake 3.14.
# The ConigVersion.cmake file contains checks for the architecture is was
# generated on, which can cause problems for header only libraries
# used with e.g. the Conan package manager. Since it is header only we
# can/should omit those checks.
set(CPPZMQ_SIZEOF_VOID_P ${CMAKE_SIZEOF_VOID_P})
set(CMAKE_SIZEOF_VOID_P "") # a simple unset is not sufficient
write_basic_package_version_file(${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
                                 VERSION ${CPPZMQ_VERSION}
                                 COMPATIBILITY AnyNewerVersion)
set(CMAKE_SIZEOF_VOID_P ${CPPZMQ_SIZEOF_VOID_P})

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cppzmq.pc.in
               ${CMAKE_CURRENT_BINARY_DIR}/cppzmq.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/cppzmq.pc
              DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

install(EXPORT ${PROJECT_NAME}-targets
        FILE ${PROJECT_NAME}Targets.cmake
        DESTINATION ${CPPZMQ_CMAKECONFIG_INSTALL_DIR})
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
              ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
              DESTINATION ${CPPZMQ_CMAKECONFIG_INSTALL_DIR})
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/libzmq-pkg-config/FindZeroMQ.cmake
              DESTINATION ${CPPZMQ_CMAKECONFIG_INSTALL_DIR}/libzmq-pkg-config)

option(CPPZMQ_BUILD_TESTS "Whether or not to build the tests" ON)

if (CPPZMQ_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
    if (CMAKE_CXX_STANDARD AND NOT CMAKE_CXX_STANDARD EQUAL 98 AND CMAKE_CXX_STANDARD GREATER_EQUAL 11)
        add_subdirectory(examples)
    endif()
endif()
