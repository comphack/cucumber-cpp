cmake_minimum_required(VERSION 2.8.12)

project(Cucumber-Cpp)

set(CUKE_USE_STATIC_BOOST ${MSVC} CACHE BOOL "Statically link Boost (except boost::test)")
set(CUKE_DISABLE_BOOST_TEST OFF CACHE BOOL "Disable boost:test")
set(CUKE_DISABLE_CPPSPEC OFF CACHE BOOL "Disable CppSpec")
set(CUKE_DISABLE_GTEST OFF CACHE BOOL "Disable Google Test framework")
set(CUKE_DISABLE_UNIT_TESTS OFF CACHE BOOL "Disable unit tests")
set(CUKE_DISABLE_E2E_TESTS OFF CACHE BOOL "Disable end-to-end tests")
set(CUKE_ENABLE_EXAMPLES OFF CACHE BOOL "Enable the examples")
set(GMOCK_DIR "" CACHE STRING "Google Mock framework sources path (otherwise downloaded)")
set(GMOCK_VER "1.7.0" CACHE STRING "Google Mock framework version to be used")
set(VERBOSE OFF CACHE BOOL "Verbose output")

set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake/modules)

#
# Generic Compiler Flags
#

if(CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -Weffc++")
    # TODO: A better fix should handle ld's --as-needed flag
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Xlinker '--no-as-needed'")
elseif(MSVC)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DNOMINMAX") # exclude M$ min/max macros
    #set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /analyze")
endif()

#
# Boost
#

if(MSVC11)
    # Boost 1.51 fixed a bug with MSVC11
    message(STATUS "Forcing Boost 1.51+ on MSVC11")
    set(BOOST_MIN_VERSION "1.51")
else()
    set(BOOST_MIN_VERSION "1.40")
endif()

set(CUKE_CORE_BOOST_LIBS thread system regex date_time program_options)
if(NOT CUKE_DISABLE_BOOST_TEST)
    set(CUKE_TEST_BOOST_LIBS unit_test_framework)
endif()

find_package(Threads REQUIRED)

if(CUKE_USE_STATIC_BOOST)
    set(CUKE_STATIC_BOOST_LIBS ${CUKE_CORE_BOOST_LIBS})
    # "An external test runner utility is required to link with dynamic library" (Boost User's Guide)
    set(CUKE_DYNAMIC_BOOST_LIBS ${CUKE_TEST_BOOST_LIBS})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DBOOST_TEST_DYN_LINK")

    if(NOT MSVC)
        set(CUKE_EXTRA_LIBRARIES ${CUKE_EXTRA_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT})
    endif()
else()
    set(CUKE_DYNAMIC_BOOST_LIBS ${CUKE_CORE_BOOST_LIBS} ${CUKE_TEST_BOOST_LIBS})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DBOOST_ALL_DYN_LINK")
endif()

if(CUKE_STATIC_BOOST_LIBS)
    set(Boost_USE_STATIC_LIBS ON)
    find_package(Boost ${BOOST_MIN_VERSION} COMPONENTS ${CUKE_STATIC_BOOST_LIBS})
endif()

if(CUKE_DYNAMIC_BOOST_LIBS)
    set(Boost_USE_STATIC_LIBS OFF)
    find_package(Boost ${BOOST_MIN_VERSION} COMPONENTS ${CUKE_DYNAMIC_BOOST_LIBS})
endif()

if(Boost_FOUND)
    include_directories(${Boost_INCLUDE_DIRS})
    set(CUKE_EXTRA_LIBRARIES ${CUKE_EXTRA_LIBRARIES} ${Boost_THREAD_LIBRARY} ${Boost_SYSTEM_LIBRARY} ${Boost_REGEX_LIBRARY} ${Boost_DATE_TIME_LIBRARY} ${Boost_PROGRAM_OPTIONS_LIBRARY})
endif()

#
# CppSpec
#

if(NOT CUKE_DISABLE_CPPSPEC)
    find_package(CppSpec)
endif()

#
# GTest
#

if(NOT CUKE_DISABLE_GTEST)
    if("" STREQUAL "${GMOCK_DIR}")
        set(GMOCK_ROOT "${CMAKE_CURRENT_BINARY_DIR}/gmock")
        find_package(GMock REQUIRED)

        if(GMOCK_FOUND)
            set(CUKE_GTEST_LIBRARIES
                ${GTEST_BOTH_LIBRARIES}
                ${CMAKE_THREAD_LIBS_INIT}
            )
            set(CUKE_GMOCK_LIBRARIES
                ${GTEST_BOTH_LIBRARIES}
                ${GMOCK_BOTH_LIBRARIES}
                ${CMAKE_THREAD_LIBS_INIT}
            )
        endif()
    else()
        add_library(gtest IMPORTED STATIC GLOBAL)
        add_library(gmock IMPORTED STATIC GLOBAL)
        add_library(gmock_main IMPORTED STATIC GLOBAL)

        set(GMOCK_FOUND True)
        set(GTEST_INCLUDE_DIRS "${GMOCK_DIR}/include")
        set(GMOCK_PREFIX "${GMOCK_DIR}/lib/${CMAKE_STATIC_LIBRARY_PREFIX}")
        set(GMOCK_SUFFIX "${CMAKE_STATIC_LIBRARY_SUFFIX}")

        set_target_properties(gtest PROPERTIES
            IMPORTED_LOCATION "${GMOCK_PREFIX}gtest${GMOCK_SUFFIX}"
            IMPORTED_LINK_INTERFACE_LIBRARIES "${CMAKE_THREAD_LIBS_INIT}"
            INTERFACE_INCLUDE_DIRECTORIES "${GMOCK_DIR}/include"
        )

        set_target_properties(gmock PROPERTIES
            IMPORTED_LOCATION "${GMOCK_PREFIX}gmock${GMOCK_SUFFIX}"
            IMPORTED_LINK_INTERFACE_LIBRARIES "${CMAKE_THREAD_LIBS_INIT}"
            INTERFACE_INCLUDE_DIRECTORIES "${GMOCK_DIR}/include"
        )

        set_target_properties(gmock_main PROPERTIES
            IMPORTED_LOCATION "${GMOCK_PREFIX}gmock_main${GMOCK_SUFFIX}"
            IMPORTED_LINK_INTERFACE_LIBRARIES "${CMAKE_THREAD_LIBS_INIT}"
            INTERFACE_INCLUDE_DIRECTORIES "${GMOCK_DIR}/include"
        )

        function(GTEST_ADD_TESTS executable extra_args)
        if(NOT ARGN)
            message(FATAL_ERROR "Missing ARGN: Read the documentation for GTEST_ADD_TESTS")
        endif()
        if(ARGN STREQUAL "AUTO")
            # obtain sources used for building that executable
            get_property(ARGN TARGET ${executable} PROPERTY SOURCES)
        endif()
        set(gtest_case_name_regex ".*\\( *([A-Za-z_0-9]+) *, *([A-Za-z_0-9]+) *\\).*")
        set(gtest_test_type_regex "(TYPED_TEST|TEST_?[FP]?)")
        foreach(source ${ARGN})
            file(READ "${source}" contents)
            string(REGEX MATCHALL "${gtest_test_type_regex} *\\(([A-Za-z_0-9 ,]+)\\)" found_tests ${contents})
            foreach(hit ${found_tests})
              string(REGEX MATCH "${gtest_test_type_regex}" test_type ${hit})

              # Parameterized tests have a different signature for the filter
              if("x${test_type}" STREQUAL "xTEST_P")
                string(REGEX REPLACE ${gtest_case_name_regex}  "*/\\1.\\2/*" test_name ${hit})
              elseif("x${test_type}" STREQUAL "xTEST_F" OR "x${test_type}" STREQUAL "xTEST")
                string(REGEX REPLACE ${gtest_case_name_regex} "\\1.\\2" test_name ${hit})
              elseif("x${test_type}" STREQUAL "xTYPED_TEST")
                string(REGEX REPLACE ${gtest_case_name_regex} "\\1/*.\\2" test_name ${hit})
              else()
                message(WARNING "Could not parse GTest ${hit} for adding to CTest.")
                continue()
              endif()
              add_test(NAME ${test_name} COMMAND ${executable} --gtest_filter=${test_name} ${extra_args})
            endforeach()
        endforeach()
    endfunction()
    endif("" STREQUAL "${GMOCK_DIR}")
endif()

#
# Cucumber-Cpp
#

set(CUKE_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/include)

include_directories(${CUKE_INCLUDE_DIR})

set(CUKE_LIBRARIES cucumber-cpp ${CUKE_EXTRA_LIBRARIES})

add_subdirectory(src)

#
# Tests
#

if(CUKE_DISABLE_UNIT_TESTS)
    message(STATUS "Skipping unit tests")
else()
    enable_testing()
    add_subdirectory(tests)
endif()

if(CUKE_DISABLE_E2E_TESTS)
    message(STATUS "Skipping end-to-end tests")
else()
    find_program(BUNDLE bundle)
    if(BUNDLE)
        message(STATUS "Installing gem dependencies")
        execute_process(COMMAND ${BUNDLE} WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
    else()
        message(WARNING "Could not find Bundler: skipping Ruby Gem management")
    endif()

    find_program(CUCUMBER_RUBY cucumber)
    if(CUCUMBER_RUBY)
        message(STATUS "Found Cucumber")
        set(CUKE_FEATURES_DIR ${CMAKE_SOURCE_DIR}/features)
        set(CUKE_FEATURES_TMP ${CMAKE_BINARY_DIR}/tmp)
        set(CUKE_TEST_FEATURES_DIR ${CUKE_FEATURES_TMP}/test_features)
        set(CUKE_DYNAMIC_CPP_STEPS ${CUKE_TEST_FEATURES_DIR}/step_definitions/cpp_steps.cpp)
        string(REPLACE "/tmp" "${CMAKE_FILES_DIRECTORY}/e2e-steps.dir/tmp" CUKE_DYNAMIC_CPP_STEPS_OBJ "${CUKE_DYNAMIC_CPP_STEPS}${CMAKE_CXX_OUTPUT_EXTENSION}")

        file(WRITE ${CUKE_DYNAMIC_CPP_STEPS})
        add_executable(e2e-steps EXCLUDE_FROM_ALL ${CUKE_DYNAMIC_CPP_STEPS})
        target_link_libraries(e2e-steps ${CUKE_LIBRARIES})

        # TODO It does not escape paths
        set(CUKE_COMPILE_DYNAMIC_CPP_STEPS "${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target e2e-steps")

        function(add_feature_target TARGET_NAME)
            add_custom_target(${TARGET_NAME}
                COMMAND ${CUCUMBER_RUBY}
                TEST_FEATURES_DIR=${CUKE_TEST_FEATURES_DIR}
                TMP_DIR=${CUKE_FEATURES_TMP}
                DYNAMIC_CPP_STEPS_SRC=${CUKE_DYNAMIC_CPP_STEPS}
                DYNAMIC_CPP_STEPS_EXE=${CMAKE_BINARY_DIR}/e2e-steps
                DYNAMIC_CPP_STEPS_OBJ=${CUKE_DYNAMIC_CPP_STEPS_OBJ}
                COMPILE_DYNAMIC_CPP_STEPS=${CUKE_COMPILE_DYNAMIC_CPP_STEPS}
                ${ARGV1} ${ARGV2} ${ARGV3} ${ARGV4} ${ARGV5} ${ARGV6}
                ${CUKE_FEATURES_DIR}
                DEPENDS cucumber-cpp
            )
        endfunction(add_feature_target)

        add_feature_target(features --format progress)
        add_feature_target(features-pretty --format pretty)
        add_feature_target(features-wip --format pretty --tags @wip)

    else()
        message(WARNING "Could not find Cucumber: skipping end-to-end tests")
    endif()

endif()

#
# Examples
#

if(CUKE_ENABLE_EXAMPLES)
    add_subdirectory(examples)
endif()

install(DIRECTORY include/cucumber-cpp
    DESTINATION include
)

install(FILES
    include/json_spirit/json_spirit_utils.h
    include/json_spirit/json_spirit_value.h
    include/json_spirit/json_spirit_writer.h
    include/json_spirit/json_spirit_stream_reader.h
    include/json_spirit/json_spirit_reader.h
    include/json_spirit/json_spirit_reader_template.h
    include/json_spirit/json_spirit.h
    include/json_spirit/json_spirit_writer_template.h
    include/json_spirit/json_spirit_error_position.h

    DESTINATION include/json_spirit
)
