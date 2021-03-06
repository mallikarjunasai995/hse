cmake_minimum_required(VERSION 3.2)
if(${CMAKE_MAJOR_VERSION} GREATER 2)
    # CMP0046 -- We need this policy because this project adds
    #            dependencies on mpool, but has no rule to build mpool
    #            (because it's not part of this project).  We shoud
    #            fix that and eliminate this policy setting.
    cmake_policy(SET CMP0046 OLD)
endif()

project( hse )

message(STATUS "Configuring ${CMAKE_PROJECT_NAME}")

set (CMAKE_C_STANDARD 99)

set( HSE_SUB_REPOS      ${PROJECT_SOURCE_DIR}/sub)
set( HSE_SCRIPT_DIR     ${PROJECT_SOURCE_DIR}/scripts)
set( HSE_STAGE_DIR      ${PROJECT_BINARY_DIR}/stage)

# CPACK_PACKAGING_INSTALL_PREFIX is used by CPack.
# CMAKE_INSTALL_PREFIX is used by:
#   - make install
#   - Cmake's "install()" function when its DESTINATION parameter
#     is a relative path.
set( CPACK_PACKAGING_INSTALL_PREFIX  /usr)
set( CMAKE_INSTALL_PREFIX            /usr)
set( CMAKE_RUNTIME_OUTPUT_DIRECTORY  ${HSE_STAGE_DIR}/bin)

# Install destinations
set( HSE_DIAG_BIN  /opt/hse/bin)
set( HSE_DOCS_DIR  share/doc/hse/)
set( HSE_UDOC_DIR  /opt/hse/doc)

set( HSE_UTIL_INCLUDE_DIRS
  ${PROJECT_SOURCE_DIR}/src/util/include
  ${CMAKE_BINARY_DIR}/stage/gensrc)

set( HSE_INCLUDE_DIRS ${PROJECT_SOURCE_DIR}/include)

set( HSE_UT_FMWK_INCLUDE_DIRS   ${PROJECT_SOURCE_DIR}/test/framework/include)
set( HSE_TEST_LIB_INCLUDE_DIRS    ${PROJECT_SOURCE_DIR}/test/library/include)
set( HSE_3RD_PARTY_INCLUDE_DIRS  ${PROJECT_SOURCE_DIR}/3rdparty/include)

include_directories(${PROJECT_SOURCE_DIR}/3rdparty/include)
include_directories(${HSE_STAGE_DIR}/include)

# increasing warning levels
set( CFLAGS_WARN_1 "${CFLAGS_WARN_0} -Wall -Werror -std=gnu99")

if ( "${HSE_DISTRO}" STREQUAL  "el6" )
   set( CFLAGS_WARN_2 "${CFLAGS_WARN_1} -Wlogical-op -Wno-missing-field-initializers  -Wuninitialized -Wno-format-zero-length")
else()
   set( CFLAGS_WARN_2 "${CFLAGS_WARN_1} -Wlogical-op -Wno-missing-field-initializers  -Wuninitialized -Wmaybe-uninitialized ")
endif()

set( CFLAGS_WARN_3 "${CFLAGS_WARN_2} -Wextra -Wno-conversion -Wno-sign-conversion -Wno-sign-compare -Wno-unused-parameter")
set( CFLAGS_WARN_4 "${CFLAGS_WARN_2} -Wextra -Wconversion    -Wsign-conversion    -Wsign-compare    -Wunused-parameter")
set( CFLAGS_WARN_MAX "${CFLAGS_WARN_4}")

set( CMAKE_C_FLAGS "${CFLAGS_WARN_3} -g3")

if ( ${UBSAN} )
    message(STATUS "Enabling ubsan support")
    add_compile_options( -fno-omit-frame-pointer )
    add_compile_options( -fsanitize=undefined -fno-sanitize=alignment )
    link_libraries( -fsanitize=undefined -fno-sanitize=alignment )
endif()

if ( ${ASAN} )
    message(STATUS "Enabling asan/lsan support")
    add_compile_options( -fno-omit-frame-pointer )
    add_compile_options( -fsanitize=address )
    link_libraries( -fsanitize=address )
endif()

# Define valgrind wrapper functions if valgrind-devel package
# is installed (sudo dnf install valgrind-devel).
find_path(ValgrindIncludes valgrind.h PATHS /usr/include/valgrind)
if(ValgrindIncludes)
    message(STATUS "Enabling valgrind support")
else()
    message(STATUS "Disabling valgrind support")
    add_definitions( -DNVALGRIND )
endif()

if( EXISTS /usr/libexec/platform-python )
    set( HSE_PYTHON /usr/libexec/platform-python )
elseif( EXISTS /etc/fedora-release )
    set( HSE_PYTHON /usr/bin/python3 )
else()
    set( HSE_PYTHON /usr/bin/python )
endif()

execute_process( COMMAND ${HSE_PYTHON} -c "from distutils.sysconfig import get_python_lib; print(get_python_lib())" OUTPUT_STRIP_TRAILING_WHITESPACE OUTPUT_VARIABLE HSE_PYTHON_PKG_DIR )

get_filename_component( HSE_PYTHON_BASE_DIR ${HSE_PYTHON_PKG_DIR}/.. ABSOLUTE )
set ( HSE_PYTHON_SOS_DIR ${HSE_PYTHON_PKG_DIR}/sos )
set ( HSE_PYTHON_SOS_PLUGINS_DIR ${HSE_PYTHON_PKG_DIR}/sos/plugins )


include(CMakeLists.macros)


# Enable the following flags to try and catch "use before initialization"
# problems that gcc doesn't catch with default build flags.
#
#set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O1 -fno-tree-ccp")


#
# Flags to enable the use of gprof
#
set( PROFILING false )

if( ${PROFILING} )
  set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pg " )
  set( CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -L /usr/local/lib -lprofiler" )
endif( ${PROFILING} )

if( HSE_UNIT_TEST_FLAG )
  message( STATUS "Compiling with unit test flag" )

  # Tool to pre-process files for unit test mocking
  set( UTPP ${HSE_SCRIPT_DIR}/build/utpp )

  # defines the include directories needed by all tests using mocking
  set( HSE_UT_BASE_INCLUDE_DIRS ${HSE_UT_FMWK_INCLUDE_DIRS}
                                ${HSE_TEST_LIB_INCLUDE_DIRS}
                                ${HSE_TEST_SUPPORT_UT_GS_INCLUDE_DIRS} )

  set( HSE_TEST_SUPPORT_LIBS hse_test_support-lib )

else()

  # defines the include directories needed by all tests using mocking
  set( HSE_UT_BASE_INCLUDE_DIRS ${HSE_UT_FMWK_INCLUDE_DIRS}
                                ${HSE_TEST_LIB_INCLUDE_DIRS} )

endif()


# Needed for the JNI bridge.
#    make -C ~/hse config CMAKE_FLAGS=-DYCSB:BOOL=TRUE
#
set( YCSB TRUE CACHE BOOL "Build JNI bridge artifacts" )
if( ${YCSB} )
    add_compile_options( -fPIC )
endif()

# Keep these lines--we need them in release branches
set( HSE_VERSION_MAJOR 0 )
set( HSE_VERSION_MINOR 0 )
set( HSE_VERSION_PATCH 0 )

#
# Set-up what kind of build this is
#
if( "${CMAKE_BUILD_TYPE}" STREQUAL "Debug" )
  add_definitions( -DHSE_BUILD_DEBUG )
  add_definitions( -DHSE_DEBUG_BUILD ) # deprecated
  add_definitions( -DHSE_LOG_PRI_DEFAULT=7 )
  add_definitions( -DDEBUG_RCU )
  set( CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fstack-protector-all")
  set( HSE_BUILD_TYPE_FLAG "d" )

elseif( "${CMAKE_BUILD_TYPE}" STREQUAL "OptDebug" )
  add_definitions( -DHSE_BUILD_DEBUG )
  add_definitions( -DHSE_LOG_PRI_DEFAULT=7 )
  set( HSE_BUILD_TYPE_FLAG "od" )

elseif( "${CMAKE_BUILD_TYPE}" STREQUAL "Release" )
  add_definitions( -DHSE_BUILD_RELEASE )
  add_definitions( -DHSE_LOG_PRI_DEFAULT=5 )
  set( HSE_BUILD_TYPE_FLAG "r" )

elseif( "${CMAKE_BUILD_TYPE}" STREQUAL "RelAssert" )
  add_definitions( -DHSE_BUILD_RELASSERT )
  add_definitions( -DHSE_LOG_PRI_DEFAULT=5 )
  set( HSE_BUILD_TYPE_FLAG "ra" )

elseif( "${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo" )
  add_definitions( -DHSE_BUILD_RELEASE )
  add_definitions( -DHSE_LOG_PRI_DEFAULT=5 )
  set( HSE_BUILD_TYPE_FLAG "rd" )

else()
  message( FATAL_ERROR "Invalid build type [${CMAKE_BUILD_TYPE}]" )
endif()

################################################################
#
# Create the version header file
#
################################################################

find_package(Git)

# Create our template for the header file
file(WRITE ${CMAKE_BINARY_DIR}/hse_version.h.in
"static const char hse_version[] = \"@HSE_VERSION_RELEASE@\";\n"
"static const char hse_branch[] = \"@HSE_BRANCH@\";\n"
"static const char hse_tag[] = \"@HSE_TAG@\";\n"
"static const char hse_buildhost[] = \"@HSE_BUILDHOST@\";\n"
)

# Create cmake code to transform input template into target header file.
file(WRITE ${CMAKE_BINARY_DIR}/hse_version.cmake
"
set(HSE_SHA ${HSE_SHA})
if( NOT DEFINED HSE_SHA )
    execute_process(
        COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
        WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
        OUTPUT_VARIABLE HSE_SHA
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
endif()
execute_process(
    COMMAND ${GIT_EXECUTABLE} describe --dirty --always --tags
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    OUTPUT_VARIABLE HSE_TAG
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
execute_process(
    COMMAND ${GIT_EXECUTABLE} rev-parse --abbrev-ref HEAD
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
    OUTPUT_VARIABLE HSE_BRANCH
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
execute_process(
    COMMAND date +%Y%m%d
    OUTPUT_VARIABLE HSE_DATESTAMP
    OUTPUT_STRIP_TRAILING_WHITESPACE
)
execute_process(
    COMMAND hostname -f
    OUTPUT_VARIABLE HSE_BUILDHOST
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

# cmake DIST doesn't work in 3.6.2.
execute_process(
    COMMAND rpm --eval %{dist}
    OUTPUT_VARIABLE PACKAGE_DIST
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

# In 3.6.x, for some reason, CPACK_RPM_PACKAGE_ARCHITECTURE is an
# empty string when this CMakeLists.txt gets processed.
execute_process(
    COMMAND uname -m
    OUTPUT_VARIABLE PACKAGE_ARCH
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

if( \${HSE_TAG} MATCHES \"dirty$\" )
    set( HSE_DIRTYTAG \".dirty\" )
endif( )

set( HSE_MAJORMINORPATCH ${HSE_VERSION_MAJOR}.${HSE_VERSION_MINOR}.${HSE_VERSION_PATCH} )

set( HSE_RELEASE
    ${HSE_BUILD_TYPE_FLAG}${BUILD_NUMBER}.\${HSE_DATESTAMP}.\${HSE_SHA}\${HSE_DIRTYTAG} )

set( HSE_VERSION \${HSE_MAJORMINORPATCH} )
set( HSE_VERSION_RELEASE \${HSE_MAJORMINORPATCH}-\${HSE_RELEASE} )

if (REL_CANDIDATE)
 set( HSE_RELEASE
      ${HSE_BUILD_TYPE_FLAG}${BUILD_NUMBER}.\${HSE_DATESTAMP} )
endif()

if( NOT HSE_VERSION_CMAKE_INCLUDEONLY )
    if (EXISTS \${DST})
        configure_file(\${SRC} \${CMAKE_BINARY_DIR}/hse_version.h.tmp @ONLY)
        file(SHA256 \${CMAKE_BINARY_DIR}/hse_version.h.tmp NEWSHA)
        file(SHA256 \${DST} OLDSHA)
        # Replace hse_version.h file with a newer only if the content changed.
        if (NOT \${OLDSHA} EQUAL \${NEWSHA})
                configure_file(\${SRC} \${DST} @ONLY)
        endif()
    else()
        configure_file(\${SRC} \${DST} @ONLY)
    endif()
endif( )
"
)

# Set up custom target
add_custom_target(
    hse_version
    COMMAND ${CMAKE_COMMAND}
        -D SRC=${CMAKE_BINARY_DIR}/hse_version.h.in
        -D DST=${HSE_STAGE_DIR}/include/hse_version.h
        -P ${CMAKE_BINARY_DIR}/hse_version.cmake
)


################################################################
#
# Collate the input files, and recreate the mapi output file
#
################################################################

file(WRITE ${CMAKE_BINARY_DIR}/mapi_idx.cmake
"
set( ENV{HSE_BUILD_TYPE_FLAG} ${HSE_BUILD_TYPE_FLAG} )
set( ENV{CMAKE_BINARY_DIR} ${CMAKE_BINARY_DIR} )
set( ENV{CMAKE_STAGE_DIR} ${HSE_STAGE_DIR} )
set( ENV{PUBLIC_HEADERS} ${MPOOL_INCLUDE_DIR} )
execute_process(
    COMMAND ${PROJECT_SOURCE_DIR}/scripts/build/mapi_idx_generate
    OUTPUT_VARIABLE CFL_OUTPUT
    OUTPUT_STRIP_TRAILING_WHITESPACE
    ERROR_VARIABLE CFL_ERROR
    ERROR_STRIP_TRAILING_WHITESPACE
    RESULT_VARIABLE CMD_RES
    WORKING_DIRECTORY ${PROJECT_SOURCE_DIR}
)
if( NOT \${CMD_RES} STREQUAL 0 )
  message( FATAL_ERROR \"Error running mapi_idx_generate.\" \"\${CFL_ERROR}\")
endif()
"
)

add_custom_target(
  mapi_idx
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_BINARY_DIR}/mapi_idx.cmake
)

################################################################
#
# Establish prerequisites for all compiled targets
#
################################################################

# All compiled binaries should depend upon hse_version and mapi_idx
macro(add_library _target)
    _add_library(${_target} ${ARGN})
    add_dependencies(${_target} hse_version mapi_idx)
endmacro()

macro(add_executable _target)
    _add_executable(${_target} ${ARGN})
    add_dependencies(${_target} hse_version mapi_idx)
endmacro()

################################################################
#
# Standard sets of libraries for applications to link against
#
################################################################

set(HSE_USER_MPOOL_LINK_LIBS
    mpool
)

#
# Needed to handle the hard-coded link line for hse_platform_init
#
set( HSE_UTIL_BIN_DIR "${PROJECT_BINARY_DIR}/src/util/src" )
set( CMAKE_EXE_LINKER_FLAGS
  "${CMAKE_EXE_LINKER_FLAGS} -L ${HSE_UTIL_BIN_DIR}" )

#
# Needed to handle the hard-coded link line for hse_kvdb_init
#
set( HSE_KVDB_BIN_DIR "${PROJECT_BINARY_DIR}/src" )
set( CMAKE_EXE_LINKER_FLAGS
  "${CMAKE_EXE_LINKER_FLAGS} -L ${HSE_KVDB_BIN_DIR}" )

################################################################
#
# Subdirectories
#
################################################################

enable_testing()

add_subdirectory( sub )
# we need to compile the test support before any component
add_subdirectory( test/library )
add_subdirectory( src )
add_subdirectory( include )
add_subdirectory( cli )
add_subdirectory( test )
add_subdirectory( scripts/package/sos )
add_subdirectory( samples )
add_subdirectory( config )


# NOTE: The package name and version values are configured at "make config"
# time.  If changes are made in the tree (thereby changing the last commit SHA),
# the configured package name won't change until the next make config.
#
# This *should* be OK for a while, since we expect our build server to clone a
# a fresh tree before every build.

set( HSE_VERSION_CMAKE_INCLUDEONLY "ON" )
include( ${CMAKE_BINARY_DIR}/hse_version.cmake )
set( HSE_VERSION_CMAKE_INCLUDEONLY "OFF" )

#
# Package options
#

set( CPACK_GENERATOR "RPM" )
set( CPACK_PACKAGE_NAME "hse" )
set( CPACK_PACKAGE_VERSION_MAJOR "${HSE_VERSION_MAJOR}" )
set( CPACK_PACKAGE_VERSION_MINOR "${HSE_VERSION_MINOR}" )
set( CPACK_PACKAGE_VERSION_PATCH "${HSE_VERSION_PATCH}" )

# Explicitly set CPACK_PACKAGE_VERSION first, otherwise CMake may generate
# nonsense version strings within generated spec files

set( CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}" )


set( CPACK_RPM_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION}" )
set( CPACK_RPM_PACKAGE_RELEASE "${HSE_RELEASE}${PACKAGE_DIST}" )
set( CPACK_RPM_PACKAGE_RELOCATABLE "ON" )
set( CPACK_RPM_RELOCATION_PATHS /usr /opt /etc )
set( CPACK_RPM_COMPONENT_INSTALL "ON" )
set( CPACK_RPM_PACKAGE_URL "https://github.com/hse-project/hse" )
set( CPACK_RPM_PACKAGE_LICENSE "ASL 2.0" )
set( CPACK_RPM_PACKAGE_GROUP "Unspecified" )
set( CPACK_RPM_PACKAGE_VENDOR "Unspecified" )


#
# We don't want to claim ownership of these directories, lest there be
# conflicts.
#
set( CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION
  /usr/lib
  /usr/lib/hse
  /usr/lib/hse/modules
  /usr/lib/sysctl.d
  ${HSE_PYTHON_BASE_DIR}
  ${HSE_PYTHON_PKG_DIR}
  ${HSE_PYTHON_SOS_DIR}
  ${HSE_PYTHON_SOS_PLUGINS_DIR}
  /etc
  /lib
  /opt
  /opt/hse
  /opt/hse/config
)

# Setting this keeps CMake 3.7+ from generating nonsense filenames
set( CPACK_RPM_FILE_NAME "RPM-DEFAULT" )

#
# How to generate the RPM spec file template for your version of CMake, so it
# can be passed to CPACK_RPM_RUNTIME_USER_BINARY_SPECFILE:
#
#   cd builds/release.fc25  # (or whatever build/dist you're working on)
#   cpack -D CPACK_RPM_GENERATE_USER_BINARY_SPECFILE_TEMPLATE=1 -G RPM
#
# https://cmake.org/cmake/help/latest/cpack_gen/rpm.html#variable:CPACK_RPM_GENERATE_USER_BINARY_SPECFILE_TEMPLATE
#

#
# Runtime
#
set( CPACK_RPM_RUNTIME_PACKAGE_NAME "${CPACK_PACKAGE_NAME}" )
set( CPACK_RPM_RUNTIME_PACKAGE_SUMMARY "Heterogeneous-memory Storage Engine (HSE) runtime" )
set( CPACK_RPM_RUNTIME_PACKAGE_DESCRIPTION "Heterogeneous-memory Storage Engine (HSE) runtime" )

# [HSE_REVISIT] CPack does not support changing the RPM release string per component -
#              this is the only reason we need to set CPACK_RPM_RUNTIME_USER_BINARY_SPECFILE
set( CPACK_RPM_RUNTIME_PACKAGE_RELEASE "${CPACK_RPM_PACKAGE_RELEASE}")
set( CPACK_RPM_RUNTIME_FILE_NAME "${CPACK_RPM_RUNTIME_PACKAGE_NAME}-${CPACK_RPM_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}.${PACKAGE_ARCH}.rpm" )

set( CPACK_RPM_RUNTIME_USER_BINARY_SPECFILE "${HSE_SCRIPT_DIR}/package/hse.spec.in")
set( CPACK_RPM_RUNTIME_PRE_INSTALL_SCRIPT_FILE "${HSE_SCRIPT_DIR}/package/rpm-pre-install.sh" )
set( CPACK_RPM_RUNTIME_POST_INSTALL_SCRIPT_FILE "${HSE_SCRIPT_DIR}/package/rpm-post-install.sh" )
set( CPACK_RPM_RUNTIME_PRE_UNINSTALL_SCRIPT_FILE "${HSE_SCRIPT_DIR}/package/rpm-pre-uninstall.sh" )
set( CPACK_RPM_RUNTIME_POST_UNINSTALL_SCRIPT_FILE "${HSE_SCRIPT_DIR}/package/rpm-post-uninstall.sh" )
set( CPACK_RPM_RUNTIME_USER_FILELIST
)
set( CPACK_RPM_RUNTIME_PACKAGE_REQUIRES "sos >= 3.2" )
#set( CPACK_COMPONENT_RUNTIME_DEPENDS "libs" )

#
# Libs - for when we build shared libs.
#
#set( CPACK_RPM_LIBS_PACKAGE_NAME "${CPACK_PACKAGE_NAME}-libs" )
#set( CPACK_RPM_LIBS_PACKAGE_FILE_NAME "${CPACK_RPM_LIBS_PACKAGE_NAME}-${CPACK_RPM_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}.${RPM_ARCH}" )

#
# Devel/SDK - currently contains static libs--see "Libs" above.
#
set( CPACK_RPM_DEVEL_PACKAGE_NAME "${CPACK_PACKAGE_NAME}-devel" )
set( CPACK_RPM_DEVEL_PACKAGE_SUMMARY "Heterogeneous-memory Storage Engine (HSE) software development kit")
set( CPACK_RPM_DEVEL_PACKAGE_DESCRIPTION "Heterogeneous-memory Storage Engine (HSE) software development kit")
set( CPACK_RPM_DEVEL_FILE_NAME "${CPACK_RPM_DEVEL_PACKAGE_NAME}-${CPACK_RPM_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}.${PACKAGE_ARCH}.rpm" )
set( CPACK_RPM_DEVEL_POST_INSTALL_SCRIPT_FILE "${HSE_SCRIPT_DIR}/package/rpm-devel-post-install.sh" )
set( CPACK_RPM_DEVEL_USER_BINARY_SPECFILE "${HSE_SCRIPT_DIR}/package/hse-devel.spec.in")
#set( CPACK_COMPONENT_DEVEL_DEPENDS "libs" )

#
# Tests - we won't want to ship our test applications in production.
#
set( CPACK_RPM_TEST_PACKAGE_NAME "${CPACK_PACKAGE_NAME}-test" )
set( CPACK_RPM_TEST_PACKAGE_RELEASE "${CPACK_RPM_PACKAGE_RELEASE}")
set( CPACK_RPM_TEST_PACKAGE_SUMMARY "Heterogeneous-memory Storage Engine (HSE) tests" )
set( CPACK_RPM_TEST_PACKAGE_DESCRIPTION "Heterogeneous-memory Storage Engine (HSE) tests" )
set( CPACK_RPM_TEST_FILE_NAME "${CPACK_RPM_TEST_PACKAGE_NAME}-${CPACK_RPM_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}.${PACKAGE_ARCH}.rpm" )
# [HSE_REVISIT] CPack does not support changing the RPM release per component -
#              this is the only reason we need to set CPACK_RPM_RUNTIME_USER_BINARY_SPECFILE
set( CPACK_RPM_TEST_USER_BINARY_SPECFILE "${HSE_SCRIPT_DIR}/hse-test.spec.in")
set( CPACK_COMPONENT_TEST_DEPENDS "runtime" )
#set( CPACK_COMPONENT_TEST_DEPENDS "libs" )

#
# Tools - stuff we want for development and customer support, but
# maybe not in the final runtime package.
#
#set( CPACK_RPM_TOOLS_PACKAGE_NAME "${CPACK_PACKAGE_NAME}-tools" )
#set( CPACK_RPM_TOOLS_FILE_NAME "${CPACK_RPM_TOOLS_PACKAGE_NAME}-${CPACK_RPM_PACKAGE_VERSION}-${CPACK_RPM_PACKAGE_RELEASE}.${PACKAGE_ARCH}.rpm" )
#set( CPACK_COMPONENT_TOOLS_DEPENDS "runtime" )
#set( CPACK_COMPONENT_TEST_DEPENDS "libs" )
include( CPack )

get_directory_property( DirIncs INCLUDE_DIRECTORIES )
get_directory_property( DirDefs COMPILE_DEFINITIONS )
get_directory_property( DirOpts COMPILE_OPTIONS )

message(STATUS "Compiler flags:")
message(STATUS "  INCLUDE_DIRECTORIES    = ${DirIncs}")
message(STATUS "  COMPILE_DEFINITIONS    = ${DirDefs}")
message(STATUS "  COMPILE_OPTIONS        = ${DirOpts}")
message(STATUS "  CMAKE_C_FLAGS          = ${CMAKE_C_FLAGS}")
message(STATUS "Add the following compiler flags based on build type:")
message(STATUS "  CMAKE_C_FLAGS_DEBUG          = ${CMAKE_C_FLAGS_DEBUG}")
message(STATUS "  CMAKE_C_FLAGS_RELEASE        = ${CMAKE_C_FLAGS_RELEASE}")
message(STATUS "  CMAKE_C_FLAGS_RELASSERT      = ${CMAKE_C_FLAGS_RELASSERT}")
message(STATUS "  CMAKE_C_FLAGS_RELWITHDEBINFO = ${CMAKE_C_FLAGS_RELWITHDEBINFO}")
message(STATUS "  CMAKE_C_FLAGS_OPTDEBUG       = ${CMAKE_C_FLAGS_OPTDEBUG}")
message(STATUS "  CMAKE_KC_FLAGS               = ${CMAKE_KC_FLAGS}")
message(STATUS "Linker flags:")
message(STATUS "  CMAKE_EXE_LINKER_FLAGS = ${CMAKE_EXE_LINKER_FLAGS}")
message(STATUS "Notes:")
message(STATUS "  * Build with: 'make VERBOSE=1' to verify compiler flags")
