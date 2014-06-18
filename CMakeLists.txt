########################################################################
# CMake build script for Google Test.
#
# To run the tests for Google Test itself on Linux, use 'make test' or
# ctest.  You can select which tests to run using 'ctest -R regex'.
# For more options, run 'ctest --help'.

# BUILD_SHARED_LIBS is a standard CMake variable, but we declare it here to
# make it prominent in the GUI.
option(BUILD_SHARED_LIBS "Build shared libraries (DLLs)." OFF)

# When other libraries are using a shared version of runtime libraries,
# Google Test also has to use one.
option(
  gtest_force_shared_crt
  "Use shared (DLL) run-time lib even when Google Test is built as static lib."
  OFF)

option(gtest_build_tests "Build all of gtest's own tests." OFF)

option(gtest_build_samples "Build gtest's sample programs." OFF)

option(gtest_disable_pthreads "Disable uses of pthreads in gtest." OFF)

# Defines pre_project_set_up_hermetic_build() and set_up_hermetic_build().
include(cmake/hermetic_build.cmake OPTIONAL)

if (COMMAND pre_project_set_up_hermetic_build)
  pre_project_set_up_hermetic_build()
endif()

########################################################################
#
# Project-wide settings

# Name of the project.
#
# CMake files in this project can refer to the root source directory
# as ${gtest_SOURCE_DIR} and to the root binary directory as
# ${gtest_BINARY_DIR}.
# Language "C" is required for find_package(Threads).
project(gtest CXX C)
cmake_minimum_required(VERSION 2.6.2)

set(GTEST_MAJOR_VERSION 1)
set(GTEST_MINOR_VERSION 6)
set(GTEST_PATCH_VERSION 0)
set(GTEST_VERSION
  ${GTEST_MAJOR_VERSION}.${GTEST_MINOR_VERSION}.${GTEST_PATCH_VERSION})

set(INSTALL_LIB_DIR lib CACHE PATH "Installation directory for libraries")
set(INSTALL_BIN_DIR bin CACHE PATH "Installation directory for executables")
set(INSTALL_INCLUDE_DIR include CACHE PATH
  "Installation directory for header files")
if(WIN32 AND NOT CYGWIN)
  set(DEF_INSTALL_CMAKE_DIR CMake)
else()
  set(DEF_INSTALL_CMAKE_DIR lib/CMake/GTest)
endif()
set(INSTALL_CMAKE_DIR ${DEF_INSTALL_CMAKE_DIR} CACHE PATH
  "Installation directory for CMake files")

# Make relative paths absolute (needed later on)
foreach(p LIB BIN INCLUDE CMAKE)
  set(var INSTALL_${p}_DIR)
  if(NOT IS_ABSOLUTE "${${var}}")
    set(${var} "${CMAKE_INSTALL_PREFIX}/${${var}}")
  endif()
endforeach()

if (COMMAND set_up_hermetic_build)
  set_up_hermetic_build()
endif()

# Define helper functions and macros used by Google Test.
include(cmake/internal_utils.cmake)

config_compiler_and_linker() # Defined in internal_utils.cmake.

add_definitions( -DGTEST_USE_OWN_TR1_TUPLE=1 )

# Where Google Test's .h files can be found.
include_directories(
  ${gtest_SOURCE_DIR}/include
  ${gtest_SOURCE_DIR})

# Where Google Test's libraries can be found.
link_directories(${gtest_BINARY_DIR}/src)

########################################################################
#
# Defines the gtest & gtest_main libraries.  User tests should link
# with one of them.

# Google Test libraries.  We build them using more strict warnings than what
# are used for other targets, to ensure that gtest can be compiled by a user
# aggressive about warnings.
cxx_library(gtest "${cxx_strict}" src/gtest-all.cc)
cxx_library(gtest_main "${cxx_strict}" src/gtest_main.cc)
target_link_libraries(gtest_main gtest)

install(TARGETS gtest_main gtest 
  EXPORT GTestTargets 
  RUNTIME DESTINATION "${INSTALL_BIN_DIR}" COMPONENT bin
  ARCHIVE DESTINATION "${INSTALL_LIB_DIR}" COMPONENT stlib
  PUBLIC_HEADER DESTINATION "${INSTALL_INCLUDE_DIR}/gtest"
    COMPONENT dev)

########################################################################
#
# At this point the include directories, the compiler flags and the
# main targets have been configured. We will now export all these 
# settings to GTestConfig.cmake and GTestConfigVersion.cmake, such that
# external projects can use these libraries using the find_package 
# command.
# 

# Store compile flags into variable.
file(WRITE flags_target.cc "//Fake source file to query compile flags in cmake")
cxx_executable(flags_target ${CMAKE_CURRENT_LIST_DIR} gtest_main)
get_property(CONF_CXX_FLAGS TARGET flags_target PROPERTY COMPILE_FLAGS)

# Add all targets to the build-tree export set
export(TARGETS gtest_main gtest 
  FILE "${PROJECT_BINARY_DIR}/GTestTargets.cmake")
 
# Export the package for use from the build-tree
# (this registers the build-tree with a global CMake-registry)
#export(PACKAGE GTest)
 
# Create the GTestConfig.cmake and GTestConfigVersion files
# ... for the build tree
get_directory_property(GLOBAL_INCLUDE_DIRS INCLUDE_DIRECTORIES)
set(CONF_INCLUDE_DIRS "${GLOBAL_INCLUDE_DIRS}")
configure_file(GTestConfig.cmake.in
  "${PROJECT_BINARY_DIR}/GTestConfig.cmake" @ONLY)

# ... for the install tree
file(RELATIVE_PATH REL_INCLUDE_DIR "${INSTALL_CMAKE_DIR}"
   "${INSTALL_INCLUDE_DIR}")
set(CONF_INCLUDE_DIRS "\${GTEST_CMAKE_DIR}/${REL_INCLUDE_DIR}")
configure_file(GTestConfig.cmake.in
  "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/GTestConfig.cmake" @ONLY)

# ... for both
configure_file(GTestConfigVersion.cmake.in
  "${PROJECT_BINARY_DIR}/GTestConfigVersion.cmake" @ONLY)
 
# Install the GTestConfig.cmake and GTestConfigVersion.cmake
install(FILES
  "${PROJECT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}/GTestConfig.cmake"
  "${PROJECT_BINARY_DIR}/GTestConfigVersion.cmake"
  DESTINATION "${INSTALL_CMAKE_DIR}" COMPONENT dev)
 
# Install the export set for use with the install-tree
install(EXPORT GTestTargets DESTINATION
  "${INSTALL_CMAKE_DIR}" COMPONENT dev)


########################################################################
#
# Samples on how to link user tests with gtest or gtest_main.
#
# They are not built by default.  To build them, set the
# gtest_build_samples option to ON.  You can do it by running ccmake
# or specifying the -Dbuild_gtest_samples=ON flag when running cmake.

if (gtest_build_samples)
  cxx_executable(sample1_unittest samples gtest_main samples/sample1.cc)
  cxx_executable(sample2_unittest samples gtest_main samples/sample2.cc)
  cxx_executable(sample3_unittest samples gtest_main)
  cxx_executable(sample4_unittest samples gtest_main samples/sample4.cc)
  cxx_executable(sample5_unittest samples gtest_main samples/sample1.cc)
  cxx_executable(sample6_unittest samples gtest_main)
  cxx_executable(sample7_unittest samples gtest_main)
  cxx_executable(sample8_unittest samples gtest_main)
  cxx_executable(sample9_unittest samples gtest)
  cxx_executable(sample10_unittest samples gtest)
endif()

########################################################################
#
# Google Test's own tests.
#
# You can skip this section if you aren't interested in testing
# Google Test itself.
#
# The tests are not built by default.  To build them, set the
# gtest_build_tests option to ON.  You can do it by running ccmake
# or specifying the -Dgtest_build_tests=ON flag when running cmake.

if (gtest_build_tests)
  # This must be set in the root directory for the tests to be run by
  # 'make test' or ctest.
  enable_testing()

  ############################################################
  # C++ tests built with standard compiler flags.

  cxx_test(gtest-death-test_test gtest_main)
  cxx_test(gtest_environment_test gtest)
  cxx_test(gtest-filepath_test gtest_main)
  cxx_test(gtest-linked_ptr_test gtest_main)
  cxx_test(gtest-listener_test gtest_main)
  cxx_test(gtest_main_unittest gtest_main)
  cxx_test(gtest-message_test gtest_main)
  cxx_test(gtest_no_test_unittest gtest)
  cxx_test(gtest-options_test gtest_main)
  cxx_test(gtest-param-test_test gtest
    test/gtest-param-test2_test.cc)
  cxx_test(gtest-port_test gtest_main)
  cxx_test(gtest_pred_impl_unittest gtest_main)
  cxx_test(gtest-printers_test gtest_main)
  cxx_test(gtest_prod_test gtest_main
    test/production.cc)
  cxx_test(gtest_repeat_test gtest)
  cxx_test(gtest_sole_header_test gtest_main)
  cxx_test(gtest_stress_test gtest)
  cxx_test(gtest-test-part_test gtest_main)
  cxx_test(gtest_throw_on_failure_ex_test gtest)
  cxx_test(gtest-typed-test_test gtest_main
    test/gtest-typed-test2_test.cc)
  cxx_test(gtest_unittest gtest_main)
  cxx_test(gtest-unittest-api_test gtest)

  ############################################################
  # C++ tests built with non-standard compiler flags.

  cxx_library(gtest_no_exception "${cxx_no_exception}"
    src/gtest-all.cc)
  cxx_library(gtest_main_no_exception "${cxx_no_exception}"
    src/gtest-all.cc src/gtest_main.cc)
  cxx_library(gtest_main_no_rtti "${cxx_no_rtti}"
    src/gtest-all.cc src/gtest_main.cc)

  cxx_test_with_flags(gtest-death-test_ex_nocatch_test
    "${cxx_exception} -DGTEST_ENABLE_CATCH_EXCEPTIONS_=0"
    gtest test/gtest-death-test_ex_test.cc)
  cxx_test_with_flags(gtest-death-test_ex_catch_test
    "${cxx_exception} -DGTEST_ENABLE_CATCH_EXCEPTIONS_=1"
    gtest test/gtest-death-test_ex_test.cc)

  cxx_test_with_flags(gtest_no_rtti_unittest "${cxx_no_rtti}"
    gtest_main_no_rtti test/gtest_unittest.cc)

  cxx_shared_library(gtest_dll "${cxx_default}"
    src/gtest-all.cc src/gtest_main.cc)

  cxx_executable_with_flags(gtest_dll_test_ "${cxx_default}"
    gtest_dll test/gtest_all_test.cc)
  set_target_properties(gtest_dll_test_
                        PROPERTIES
                        COMPILE_DEFINITIONS "GTEST_LINKED_AS_SHARED_LIBRARY=1")

  if (NOT MSVC OR NOT MSVC_VERSION EQUAL 1600)
    # The C++ Standard specifies tuple_element<int, class>.
    # Yet MSVC 10's <utility> declares tuple_element<size_t, class>.
    # That declaration conflicts with our own standard-conforming
    # tuple implementation.  Therefore using our own tuple with
    # MSVC 10 doesn't compile.
    cxx_library(gtest_main_use_own_tuple "${cxx_use_own_tuple}"
      src/gtest-all.cc src/gtest_main.cc)

    cxx_test_with_flags(gtest-tuple_test "${cxx_use_own_tuple}"
      gtest_main_use_own_tuple test/gtest-tuple_test.cc)

    cxx_test_with_flags(gtest_use_own_tuple_test "${cxx_use_own_tuple}"
      gtest_main_use_own_tuple
      test/gtest-param-test_test.cc test/gtest-param-test2_test.cc)
  endif()

  ############################################################
  # Python tests.

  cxx_executable(gtest_break_on_failure_unittest_ test gtest)
  py_test(gtest_break_on_failure_unittest)

  cxx_executable_with_flags(
    gtest_catch_exceptions_no_ex_test_
    "${cxx_no_exception}"
    gtest_main_no_exception
    test/gtest_catch_exceptions_test_.cc)
  cxx_executable_with_flags(
    gtest_catch_exceptions_ex_test_
    "${cxx_exception}"
    gtest_main
    test/gtest_catch_exceptions_test_.cc)
  py_test(gtest_catch_exceptions_test)

  cxx_executable(gtest_color_test_ test gtest)
  py_test(gtest_color_test)

  cxx_executable(gtest_env_var_test_ test gtest)
  py_test(gtest_env_var_test)

  cxx_executable(gtest_filter_unittest_ test gtest)
  py_test(gtest_filter_unittest)

  cxx_executable(gtest_help_test_ test gtest_main)
  py_test(gtest_help_test)

  cxx_executable(gtest_list_tests_unittest_ test gtest)
  py_test(gtest_list_tests_unittest)

  cxx_executable(gtest_output_test_ test gtest)
  py_test(gtest_output_test)

  cxx_executable(gtest_shuffle_test_ test gtest)
  py_test(gtest_shuffle_test)

  cxx_executable(gtest_throw_on_failure_test_ test gtest_no_exception)
  set_target_properties(gtest_throw_on_failure_test_
    PROPERTIES
    COMPILE_FLAGS "${cxx_no_exception}")
  py_test(gtest_throw_on_failure_test)

  cxx_executable(gtest_uninitialized_test_ test gtest)
  py_test(gtest_uninitialized_test)

  cxx_executable(gtest_xml_outfile1_test_ test gtest_main)
  cxx_executable(gtest_xml_outfile2_test_ test gtest_main)
  py_test(gtest_xml_outfiles_test)

  cxx_executable(gtest_xml_output_unittest_ test gtest)
  py_test(gtest_xml_output_unittest)
endif()
