cmake_minimum_required(VERSION 3.8.0)
# NOTE: policy for using the CheckIPOSupported module:
# https://cmake.org/cmake/help/latest/policy/CMP0069.html
cmake_policy(SET CMP0069 NEW)

# Set default build type to "Release".
# NOTE: this should be done before the project command since the latter can set
# CMAKE_BUILD_TYPE itself (it does so for nmake).
if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Release CACHE STRING
		"Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
	FORCE)
endif()

project(pygmo VERSION 2.16.0 LANGUAGES CXX C)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake" "${CMAKE_CURRENT_SOURCE_DIR}/cmake/yacma")

message(STATUS "System name: ${CMAKE_SYSTEM_NAME}")
message(STATUS "pygmo version: ${pygmo_VERSION}")

option(PYGMO_ENABLE_IPO "Enable IPO (requires CMake >= 3.9 and compiler support)." OFF)
mark_as_advanced(PYGMO_ENABLE_IPO)

# Run the YACMA compiler setup.
include(YACMACompilerLinkerSettings)

# NOTE: on Unix systems, the correct library installation path
# could be something other than just "lib", such as "lib64",
# "lib32", etc., depending on platform/configuration. Apparently,
# CMake provides this information via the GNUInstallDirs module.
# Let's enable this for now on all Unixes except OSX.
# NOTE: potentially, this could be applicable to Cygwin as well.
#
# https://cmake.org/cmake/help/v3.15/module/GNUInstallDirs.html
# https://cmake.org/pipermail/cmake/2013-July/055375.html
if(UNIX AND NOT APPLE)
    include(GNUInstallDirs)
    set(_PYGMO_INSTALL_LIBDIR_DEFAULT "${CMAKE_INSTALL_LIBDIR}")
else()
    set(_PYGMO_INSTALL_LIBDIR_DEFAULT "lib")
endif()
if(NOT PYGMO_INSTALL_LIBDIR)
    set(PYGMO_INSTALL_LIBDIR "${_PYGMO_INSTALL_LIBDIR_DEFAULT}" CACHE STRING
        "Library installation directory." FORCE)
endif()
mark_as_advanced(PYGMO_INSTALL_LIBDIR)
message(STATUS "Library installation directory: ${PYGMO_INSTALL_LIBDIR}")

# Assemble the flags.
set(PYGMO_CXX_FLAGS_DEBUG ${YACMA_CXX_FLAGS} ${YACMA_CXX_FLAGS_DEBUG})
set(PYGMO_CXX_FLAGS_RELEASE ${YACMA_CXX_FLAGS})

if(YACMA_COMPILER_IS_MSVC)
    include(CheckCXXCompilerFlag)
    # Disable the idiotic minmax macros on MSVC (both cl and clang-cl).
    # Also, enable the bigobj flag and the WIN32_LEAN_AND_MEAN definitions:
    # https://stackoverflow.com/questions/11040133/what-does-defining-win32-lean-and-mean-exclude-exactly
    list(APPEND PYGMO_CXX_FLAGS_DEBUG "-DNOMINMAX" "/bigobj" "-DWIN32_LEAN_AND_MEAN")
    list(APPEND PYGMO_CXX_FLAGS_RELEASE "-DNOMINMAX" "/bigobj" "-DWIN32_LEAN_AND_MEAN")
    # Enable strict conformance mode, if supported.
    set(CMAKE_REQUIRED_QUIET TRUE)
    check_cxx_compiler_flag("/permissive-" _PYGMO_MSVC_SUPPORTS_STRICT_CONFORMANCE)
    unset(CMAKE_REQUIRED_QUIET)
    if(_PYGMO_MSVC_SUPPORTS_STRICT_CONFORMANCE)
        message(STATUS "The '/permissive-' flag is supported, enabling it.")
        list(APPEND PYGMO_CXX_FLAGS_DEBUG "/permissive-")
        list(APPEND PYGMO_CXX_FLAGS_RELEASE "/permissive-")
    endif()
    unset(_PYGMO_MSVC_SUPPORTS_STRICT_CONFORMANCE)
    if(YACMA_COMPILER_IS_CLANGXX)
        # clang-cl emits various warnings from third party deps, let's just silence them.
        # NOTE: at one point in the recent past, MSVC added an options similar to GCC's isystem:
        # https://blogs.msdn.microsoft.com/vcblog/2017/12/13/broken-warnings-theory/
        # We probably just need to wait for this to be picked up by CMake/clang-cl. Let's
        # revisit the issue in the future.
        list(APPEND _PYGMO_CLANG_CL_DISABLED_WARNINGS
            "-Wno-unused-variable"
            "-Wno-inconsistent-dllimport"
            "-Wno-unknown-pragmas"
            "-Wno-unused-parameter"
            "-Wno-sign-compare"
            "-Wno-deprecated-declarations"
            "-Wno-deprecated-dynamic-exception-spec"
            "-Wno-old-style-cast"
            "-Wno-sign-conversion"
            "-Wno-non-virtual-dtor"
            "-Wno-deprecated"
            "-Wno-shadow"
            "-Wno-shorten-64-to-32"
            "-Wno-reserved-id-macro"
            "-Wno-undef"
            "-Wno-c++98-compat-pedantic"
            "-Wno-documentation-unknown-command"
            "-Wno-zero-as-null-pointer-constant"
            "-Wno-language-extension-token"
            "-Wno-gnu-anonymous-struct"
            "-Wno-nested-anon-types"
            "-Wno-documentation"
            "-Wno-comma"
            "-Wno-nonportable-system-include-path"
            "-Wno-global-constructors"
            "-Wno-redundant-parens"
            "-Wno-exit-time-destructors"
            "-Wno-missing-noreturn"
            "-Wno-switch-enum"
            "-Wno-covered-switch-default"
            "-Wno-float-equal"
            "-Wno-double-promotion"
            "-Wno-microsoft-enum-value"
            "-Wno-missing-prototypes"
            "-Wno-implicit-fallthrough"
            "-Wno-format-nonliteral"
            "-Wno-cast-qual"
            "-Wno-disabled-macro-expansion"
            "-Wno-unused-private-field"
            "-Wno-unused-template"
            "-Wno-unused-macros"
            "-Wno-extra-semi-stmt"
            "-Wno-c++98-compat")
        list(APPEND PYGMO_CXX_FLAGS_DEBUG ${_PYGMO_CLANG_CL_DISABLED_WARNINGS})
        list(APPEND PYGMO_CXX_FLAGS_RELEASE ${_PYGMO_CLANG_CL_DISABLED_WARNINGS})
        unset(_PYGMO_CLANG_CL_DISABLED_WARNINGS)
    else()
        # Problematic MSVC cl warnings.
        list(APPEND PYGMO_CXX_FLAGS_DEBUG "/wd4459" "/wd4251")
        list(APPEND PYGMO_CXX_FLAGS_RELEASE "/wd4459" "/wd4251")
    endif()
endif()
if(MINGW)
    # In MinGW some tests generate big object files.
    message(STATUS "Enabling the '-Wa,-mbig-obj' flag for MinGW.")
    list(APPEND PYGMO_CXX_FLAGS_DEBUG "-Wa,-mbig-obj")
    list(APPEND PYGMO_CXX_FLAGS_RELEASE "-Wa,-mbig-obj")
endif()
# NOTE: at least up to version 7, GCC is needlessly chatty
# about the 'override' attribute. Thus, we manually disable
# the -Wsuggest-override debug flag.
if(YACMA_COMPILER_IS_GNUCXX AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS "8")
    include(CheckCXXCompilerFlag)
    set(CMAKE_REQUIRED_QUIET TRUE)
    check_cxx_compiler_flag("-Wno-suggest-override" _PYGMO_GCC_SUPPORTS_NO_OVERRIDE)
    unset(CMAKE_REQUIRED_QUIET)
    if(_PYGMO_GCC_SUPPORTS_NO_OVERRIDE)
        message(STATUS "Enabling the '-Wno-suggest-override' flag for GCC<8.")
        list(APPEND PYGMO_CXX_FLAGS_DEBUG "-Wno-suggest-override")
    endif()
    unset(_PYGMO_GCC_SUPPORTS_NO_OVERRIDE)
endif()

# Find the dependencies.

# TBB.
# TODO This should actually be a submodule of pagmo.
# The only reason to have it here is to avoid forking pagmo as well.
set(TBB_BUILD_TESTS OFF CACHE BOOL "" FORCE)
set(TBB_BUILD_SHARED ON CACHE BOOL "" FORCE)
set(TBB_BUILD_STATIC OFF CACHE BOOL "" FORCE)
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/tbb EXCLUDE_FROM_ALL)
set(TBB_FOUND TRUE)
set(TBB_LIBRARIES tbb tbb_interface)
set(TBB_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/tbb/include)
add_library(TBB::tbb ALIAS tbb)
add_library(TBB::tbb_interface ALIAS tbb_interface)
install(TARGETS ${TBB_LIBRARIES} EXPORT pagmo_export
	LIBRARY DESTINATION ${CMAKE_CURRENT_BINARY_DIR}
	ARCHIVE DESTINATION ${CMAKE_CURRENT_BINARY_DIR}
	RUNTIME DESTINATION ${CMAKE_CURRENT_BINARY_DIR})

# pagmo.
# NOTE: put the minimum version in a variable
# so that we can re-use it below.
set(_PYGMO_MIN_PAGMO_VERSION 2.13.0)
option(PAGMO_BUILD_STATIC_LIBRARY "" TRUE)
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/pagmo2 EXCLUDE_FROM_ALL)

# python.
include(YACMAPythonSetup)

# Boost setup.
include(PygmoFindBoost)

# python version check.
if(${PYTHON_VERSION_MAJOR} LESS 3 OR (${PYTHON_VERSION_MAJOR} EQUAL 3 AND ${PYTHON_VERSION_MINOR} LESS 4))
    message(FATAL_ERROR "Minimum supported python version is 3.4.")
endif()

# pybind11.
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/ThirdParty/pybind11)

# Configure the sphinx config file.
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/doc/conf.py.in" "${CMAKE_CURRENT_SOURCE_DIR}/doc/conf.py" @ONLY)

# Add the module directory.
add_subdirectory(pygmo)
