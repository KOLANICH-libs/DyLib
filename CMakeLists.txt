cmake_minimum_required(VERSION 3.14)

project(dylib VERSION "2.1.0" HOMEPAGE_URL "https://github.com/martin-olivier/dylib" DESCRIPTION "C++ cross-platform wrapper around dynamic loading of shared libraries (dll, so, dylib)" LANGUAGES CXX)

if(NOT "${CMAKE_CXX_STANDARD}")
  set(CMAKE_CXX_STANDARD 11)
endif()
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_library(dylib INTERFACE)
target_include_directories(dylib INTERFACE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> $<INSTALL_INTERFACE:include>)
if(UNIX)
    target_link_libraries(dylib INTERFACE dl)
endif()

option(DYLIB_BUILD_TESTS "When set to ON, build unit tests" OFF)
option(DYLIB_WARNING_AS_ERRORS "Treat warnings as errors" OFF)

if(DYLIB_BUILD_TESTS)
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR})
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR})

    find_package(googletest QUIET)
    include(FetchContent)
    FetchContent_Declare(
        googletest
        URL https://github.com/google/googletest/archive/refs/tags/release-1.12.1.zip
    )
    set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
    FetchContent_MakeAvailable(googletest)

    if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        add_compile_options(-Wall -Wextra -Weffc++)
    elseif(CMAKE_CXX_COMPILER_ID MATCHES ".*Clang")
        add_compile_options(-Wall -Wextra)
    elseif(MSVC)
        add_compile_options(/W4)
    endif()

    if(DYLIB_WARNING_AS_ERRORS)
        if(MSVC)
            add_compile_options(/WX)
        elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES ".*Clang")
            add_compile_options(-Werror)
        endif()
    endif()

    add_library(dynamic_lib SHARED tests/lib.cpp)
    if(WIN32 AND MINGW)
        set_target_properties(dynamic_lib PROPERTIES PREFIX "")
    endif()

    enable_testing()

    if(UNIX AND NOT APPLE)
        add_compile_options(-fprofile-arcs -ftest-coverage)
    endif()

    add_executable(unit_tests tests/tests.cpp)
    add_dependencies(unit_tests dynamic_lib)
    target_link_libraries(unit_tests PRIVATE gtest_main dylib)

    if(UNIX AND NOT APPLE)
        target_link_libraries(unit_tests PRIVATE gcov)
    endif()

    include(GoogleTest)
    gtest_discover_tests(unit_tests PROPERTIES DISCOVERY_TIMEOUT 600 WORKING_DIRECTORY ${PROJECT_BINARY_DIR})
endif()

set(ADDITIONAL_MODULES_DIR "${CMAKE_CURRENT_LIST_DIR}/cmake")
list(APPEND CMAKE_MODULE_PATH "${ADDITIONAL_MODULES_DIR}" "${ADDITIONAL_MODULES_DIR}/GenPkgConfig")
message(STATUS "CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH}")
include(CPackComponent)
include(CMakePackageConfigHelpers)
include(GenPkgConfig)

string(REPLACE "/${CMAKE_LIBRARY_ARCHITECTURE}" "" CMAKE_INSTALL_LIBDIR_ARCHIND "${CMAKE_INSTALL_LIBDIR}")

set(PackagingTemplatesDir "${CMAKE_CURRENT_SOURCE_DIR}/packaging")

set(EXPORT_NAMESPACE "${PROJECT_NAME}::")

cpack_add_component(dev
	DISPLAY_NAME "${PROJECT_NAME} header"
	DESCRIPTION "The header needed for using ${PROJECT_NAME} in own software"
)

install(TARGETS "${PROJECT_NAME}"
	EXPORT ${PROJECT_NAME}
	INCLUDES
		DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
		# COMPONENT "${SDK_COMPONENT_NAME}" # component is not allowed for includes! Headers are installed separately! Includes only marks the headers for export
)

install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/include/${PROJECT_NAME}.hpp"
	TYPE INCLUDE
	COMPONENT "dev"
)


configure_pkg_config_file("${PROJECT_NAME}"
	NAME "${PROJECT_NAME}"
	VERSION "${PROJECT_VERSION}"
	DESCRIPTION "${CPACK_PACKAGE_DESCRIPTION}"
	URL "${CPACK_PACKAGE_HOMEPAGE_URL}"
	INSTALL_LIB_DIR "${CMAKE_INSTALL_LIBDIR_ARCHIND}"
	INSTALL_INCLUDE_DIR "${CMAKE_INSTALL_INCLUDEDIR}"
	COMPONENT "dev"
)

set(CMAKE_CONFIG_FILE_BASENAME "${PROJECT_NAME}Config.cmake")
set(CMAKE_EXPORT_FILE_BASENAME "${PROJECT_NAME}Export.cmake")
set(CMAKE_CONFIG_VERSION_FILE_BASENAME "${PROJECT_NAME}ConfigVersion.cmake")
set(CMAKE_CONFIG_VERSION_FILE_NAME "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CONFIG_VERSION_FILE_BASENAME}")
configure_package_config_file("${PackagingTemplatesDir}/CMakeConfig.cmake.in" "${CMAKE_CONFIG_FILE_NAME}"
	INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR_ARCHIND}/cmake"
)

set(CMAKE_CONFIG_VERSION_FILE_NAME "${CMAKE_CURRENT_BINARY_DIR}/${CMAKE_CONFIG_VERSION_FILE_BASENAME}")
write_basic_package_version_file(
	"${CMAKE_CONFIG_VERSION_FILE_NAME}"
	VERSION ${PROJECT_VERSION}
	COMPATIBILITY AnyNewerVersion
	ARCH_INDEPENDENT
)
install(FILES ${CMAKE_CONFIG_VERSION_FILE_NAME}
	DESTINATION ${CMAKE_INSTALL_LIBDIR_ARCHIND}/cmake/${PROJECT_NAME}
	COMPONENT "dev"
)

install(EXPORT "${PROJECT_NAME}"
	FILE "${CMAKE_CONFIG_FILE_BASENAME}"
	NAMESPACE "${EXPORT_NAMESPACE}"
	DESTINATION "${CMAKE_INSTALL_LIBDIR_ARCHIND}/cmake/${PROJECT_NAME}"
	COMPONENT "dev"
)

set("CPACK_DEBIAN_DEV_PACKAGE_NAME" "lib${PROJECT_NAME}-dev")
set("CPACK_DEBIAN_DEV_PACKAGE_DEPENDS" "libc6-dev")
set("CPACK_DEBIAN_DEV_PACKAGE_SUGGESTS" "cmake, pkg-config, pkg-conf")

set("CPACK_RPM_DEV_PACKAGE_NAME" "lib${PROJECT_NAME}-devel")
set("CPACK_RPM_DEV_PACKAGE_SUGGESTS" "${CPACK_DEBIAN_DEV_PACKAGE_SUGGESTS}")

set(CPACK_COMPONENTS_ALL "dev")

set(CPACK_PACKAGE_HOMEPAGE_URL "https://github.com/martin-olivier/dylib")
set(CPACK_PACKAGE_MAINTAINER "${CPACK_PACKAGE_VENDOR}")
set(CPACK_PACKAGE_CONTACT "${PROJECT_HOMEPAGE_URL}")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "${CPACK_PACKAGE_MAINTAINER}")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
set(CPACK_RESOURCE_FILE_README "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
set(CPACK_PACKAGE_MAINTAINER "${CPACK_PACKAGE_VENDOR}")
set(CPACK_DEB_COMPONENT_INSTALL ON)
set(CPACK_RPM_COMPONENT_INSTALL ON)
set(CPACK_NSIS_COMPONENT_INSTALL ON)
set(CPACK_DEBIAN_COMPRESSION_TYPE "xz")

include(CPack)
