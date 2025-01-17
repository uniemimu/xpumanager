cmake_minimum_required(VERSION 3.14.0)

if(NOT DEFINED WORK_PATH)
  set(WORK_PATH ${CMAKE_CURRENT_BINARY_DIR})
endif()

include(.cmake/xpum_version.cmake)

project(xpumanager VERSION ${PROJECT_VERSION})

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Set compilation options
include(CheckCXXCompilerFlag)
check_cxx_compiler_flag("-std=c++14" COMPILER_SUPPORTS_CXX14)
if(COMPILER_SUPPORTS_CXX14)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
else()
  message(
    STATUS
      "The compiler ${CMAKE_CXX_COMPILER} has no C++14 support.  Please use a different C++ compiler."
  )
endif()

set(CMAKE_CXX_FLAGS
    "$ENV{CXXFLAGS} -Wall -pthread -fPIC -fno-strict-overflow -fno-delete-null-pointer-checks -fwrapv"
)
set(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -g -ggdb")
set(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS} -O3")

set(CMAKE_SHARED_LINKER_FLAGS
    "${CMAKE_SHARED_LINKER_FLAGS} -z noexecstack -z relro -z now")
set(CMAKE_EXE_LINKER_FLAGS
    "${CMAKE_EXE_LINKER_FLAGS} -z noexecstack -z relro -z now")

set(CPACK_SET_DESTDIR OFF)
set(CPACK_RPM_PACKAGE_RELOCATABLE ON)

set(CPACK_XPUM_LIB_INSTALL_DIR "/usr/lib")
set(CPACK_XPUM_INCLUDE_INSTALL_DIR "/usr/include")
set(CPACK_PACKAGING_INSTALL_PREFIX "/opt/xpum")
set(SYSTEMD_UNIT_NAME "${CMAKE_CURRENT_LIST_DIR}/daemon/xpum.service")

set(XPUM_CONFIG_DIR "${CPACK_PACKAGING_INSTALL_PREFIX}/config")

set(CMAKE_PROJECT_VERSION "${PROJECT_VERSION}")
add_definitions(-DLOADER_VERSION_MAJOR=${PROJECT_VERSION_MAJOR})
add_definitions(-DLOADER_VERSION_MINOR=${PROJECT_VERSION_MINOR})
add_definitions(-DLOADER_VERSION_PATCH=${PROJECT_VERSION_PATCH})

message(STATUS "CMAKE_PROJECT_VERSION: ${CMAKE_PROJECT_VERSION}")

if(EXISTS third_party/googletest)
  add_subdirectory(third_party/googletest)
endif()
add_subdirectory(third_party/spdlog)
add_subdirectory(third_party/hwloc)
add_subdirectory(third_party/pcm)

add_subdirectory(core)
add_subdirectory(cli)
add_subdirectory(daemon)
add_subdirectory(rest)
if(BUILD_DOC)
  add_subdirectory(doc)
endif()

add_dependencies(xpum lib_hwloc)
if(EXISTS ${CMAKE_CURRENT_LIST_DIR}/core/test)
  add_dependencies(test_xpum_api lib_hwloc)
endif()

if(NOT DEFINED CPACK_GENERATOR)
  if(EXISTS "/etc/debian_version")
    set(CPACK_GENERATOR "DEB")
  elseif(EXISTS "/etc/redhat-release")
    set(CPACK_GENERATOR "RPM")
  elseif(EXISTS "/etc/SUSE-brand" OR EXISTS "/etc/SUSE-release")
    set(CPACK_GENERATOR "RPM")
  else()
    set(CPACK_GENERATOR "ZIP")
  endif()
endif()

message("CPACK_PACKAGE_NAME=${PROJECT_NAME}")
set(CPACK_PACKAGE_CONTACT "Intel Corporation")
set(CPACK_PACKAGE_NAME "${PROJECT_NAME}")
set(CPACK_PACKAGE_FILE_NAME
    "${PROJECT_NAME}.${PROJECT_VERSION}.${BUILD_TIME}.${GIT_COMMIT}")
set(CPACK_PACKAGE_VENDOR "Intel")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "XPU Manager")
set(CPACK_PACKAGE_VERSION_MAJOR "${PROJECT_VERSION_MAJOR}")
set(CPACK_PACKAGE_VERSION_MINOR "${PROJECT_VERSION_MINOR}")
set(CPACK_PACKAGE_VERSION_PATCH "${PROJECT_VERSION_PATCH}")

if(CPACK_GENERATOR MATCHES "RPM")
  configure_file("${SYSTEMD_UNIT_NAME}.in" "${SYSTEMD_UNIT_NAME}" COPYONLY)
  configure_file("${CMAKE_CURRENT_LIST_DIR}/rest/xpum_rest.service.in"
                 "${CMAKE_CURRENT_LIST_DIR}/rest/xpum_rest.service" COPYONLY)

  file(WRITE "${CMAKE_BINARY_DIR}/package_file_name"
       "${CPACK_PACKAGE_FILE_NAME}.rpm")
  set(CPACK_RPM_SPEC_MORE_DEFINE
      "%global __os_install_post %(echo '%{__os_install_post}' | sed -e 's!/usr/lib[^[:space:]]*/brp-python-bytecompile[[:space:]].*$!!g')"
  )
  set(CPACK_RPM_COMPRESSION_TYPE "xz")
  set(CPACK_RPM_PACKAGE_ARCHITECTURE "x86_64")
  set(CPACK_RPM_PACKAGE_AUTOREQ OFF)
  set(CPACK_RPM_PACKAGE_DESCRIPTION "XPU Manager")
  set(CPACK_RPM_PACKAGE_GROUP "System Environment/Libraries")
  set(CPACK_RPM_PACKAGE_LICENSE "MIT")
  set(CPACK_RPM_PACKAGE_RELEASE 1)
  set(CPACK_RPM_PACKAGE_RELEASE
      "${PROJECT_REAL_VERSION}.${BUILD_DATE}.${BUILD_TIME}.${GIT_COMMIT}")
  set(CPACK_RPM_PACKAGE_RELEASE_DIST OFF)
  set(CPACK_RPM_COMPONENT_INSTALL ON)
  set(CPACK_RPM_PACKAGE_NAME "${PROJECT_NAME}")
  list(
    APPEND
    CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION
    "/etc/ld.so.conf.d"
    "/usr/local"
    "/usr/local/lib64"
    "/usr/local/bin"
    "/usr/local/include"
    "/lib"
    "/usr/lib"
    "/usr/local/lib")

  set(CPACK_RPM_POST_INSTALL_SCRIPT_FILE
      "${CMAKE_CURRENT_LIST_DIR}/install/script/postinst")
  set(CPACK_RPM_PRE_UNINSTALL_SCRIPT_FILE
      "${CMAKE_CURRENT_LIST_DIR}/install/script/prerm")
endif()

if(CPACK_GENERATOR MATCHES "DEB")
  configure_file("${SYSTEMD_UNIT_NAME}.in" "${SYSTEMD_UNIT_NAME}" @ONLY)
  configure_file("${CMAKE_CURRENT_LIST_DIR}/rest/xpum_rest.service.in"
                 "${CMAKE_CURRENT_LIST_DIR}/rest/xpum_rest.service" @ONLY)

  file(WRITE "${CMAKE_BINARY_DIR}/package_file_name"
       "${CPACK_PACKAGE_FILE_NAME}.deb")

  set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS OFF)
  set(CPACK_DEBIAN_PACKAGE_RELEASE "${BUILD_DATE}.${BUILD_TIME}.${GIT_COMMIT}")
  set(CPACK_DEBIAN_PACKAGE_ARCHITECTURE "amd64")
  set(CPACK_DEBIAN_XPUM_PACKAGE_NAME "${PROJECT_NAME}")
  set(CPACK_DEB_COMPONENT_INSTALL ON)
  set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)

  set(CPACK_DEBIAN_PACKAGE_CONTROL_EXTRA
      "${CMAKE_CURRENT_LIST_DIR}/install/script/postinst;${CMAKE_CURRENT_LIST_DIR}/install/script/prerm;"
  )
endif()

install(DIRECTORY core/include/ DESTINATION ${CPACK_XPUM_INCLUDE_INSTALL_DIR})

install(DIRECTORY install/tools/rest/ DESTINATION .)

install(
  TARGETS xpum xpumcli xpumd
  LIBRARY DESTINATION ${CPACK_XPUM_LIB_INSTALL_DIR}
  RUNTIME DESTINATION bin
  ARCHIVE DESTINATION .)

if(EXISTS install/tools/fwUpgrade/GfxFwFPT)
  install(
    FILES install/tools/fwUpgrade/GfxFwFPT
    PERMISSIONS OWNER_READ OWNER_WRITE OWNER_EXECUTE GROUP_EXECUTE WORLD_EXECUTE
    DESTINATION /usr/local/bin)
endif()

install(FILES "${SYSTEMD_UNIT_NAME}"
              "${CMAKE_CURRENT_LIST_DIR}/rest/xpum_rest.service"
        DESTINATION ".")

include(CPack)
