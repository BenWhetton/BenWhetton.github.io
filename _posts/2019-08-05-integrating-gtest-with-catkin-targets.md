---
layout: post
title: "Integrating GTest with Catkin `tests` and `run_tests` convenience targets"
---

## Intro

ROS (robot operating system) is a fairly ubiquitous framework for writing 
robotics software. It has roots in adademia and I find many of the design 
decisions suspect from a software engineering perspective but it is pretty 
useful nonetheless. It has it’s own build system, catkin, a combination of 
python scripts and CMake macros which is intended to make it easier for people 
without a strong software engineering background to build robotics software.

A couple of the Catkin CMake macros are used to add tests. They make it 
convenient to run all the tests with a single command by adding all test CMake 
targets as a dependencies of the `tests` target and setting things up such that
 building the `run_tests` target runs it.

For various reasons, I wanted to be able to add tests such that they are 
integrated with the `tests` + `run_tests` workflow without using catkin CMake 
functions. Prior to this, I didn't really know much about CMake and this was a 
good CMake learning experience for me.

## The full solution

```cmake
# Adds an executable containing GooleTest cases and/or suites
# Integrates with Catkin `tests` and `run_tests` targets
# The test file name must be ${test_name}.cpp
# Libraries must still be linked to the new executable target
function(add_gtest test_name)
    if(TARGET gtest_main)
        set(gtest_target gtest_main)
    elseif(TARGET GTest::main)
        set(gtest_target GTest::main)
    else()
        message(FATAL_ERROR "GTest main target can't be found. Can't create test target.")
    endif()

    # Add executable and find tests
    set(results_xml ${CMAKE_BINARY_DIR}/test_results/${PROJECT_NAME}/${test_name}.xml)
    add_executable(${test_name} ${test_name}.cpp)
    target_link_libraries(${test_name} ${gtest_target})
    gtest_discover_tests(${test_name} EXTRA_ARGS --gtest_output=xml:${results_xml})

    # Set up custom target to trigger test run
    add_custom_target(
            _run_tests_${test_name}
            COMMAND ${test_name} --gtest_output=xml:${results_xml} || (exit 0))
    add_dependencies(_run_tests_${test_name} ${test_name})

    # Add Catkin aggregate test targets if they don't exist
    if(NOT TARGET tests)
        add_custom_target(tests)
    endif()
    if(NOT TARGET run_tests)
        add_custom_target(run_tests)
    endif()

    # Add dependencies to integrate with Catkin-created targets
    add_dependencies(tests ${test_name})
    add_dependencies(run_tests _run_tests_${test_name})
endfunction()
```

## How to use it

To use it, assume there is a file called `test_my_stuff.cpp` which tests the `my_big_library` target.
```cmake
add_gtest(test_my_stuff)
target_link_libraries(test_my_stuff my_big_library)
```

Now, building the `tests` target will also build the `test_my_stuff` target and 
building the `run_tests` target will run `test_my_stuff` and save the results as
 an xml file in the same `test_results` directory where tests added using Catkin
 are stored.

## Lets break it down

### Keep it simple
```cmake
# Adds an executable containing GooleTest cases and/or suites
# Integrates with Catkin `tests` and `run_tests` targets
# The test file name must be ${test_name}.cpp
# Libraries must still be linked to the new executable target
```
Firstly, just to explain, I decided not to try to do any of the library linking
inside my `add_gtest` function since it wouldn't really save much typing and
would add unnecessary complexity.

### Ensure gtest is available
```cmake
function(add_gtest test_name)
    if(TARGET gtest_main)
        set(gtest_target gtest_main)
    elseif(TARGET GTest::main)
        set(gtest_target GTest::main)
    else()
        message(FATAL_ERROR "GTest main target can't be found. Can't create test target.")
    endif()
```

This checks for the common GTest targets that I am aware of and triggers a CMake
fatal error if neither can be found.

### Create the test target
```cmake
    set(results_xml ${CMAKE_BINARY_DIR}/test_results/${PROJECT_NAME}/${test_name}.xml)
    add_executable(${test_name} ${test_name}.cpp)
    target_link_libraries(${test_name} ${gtest_target})
    gtest_discover_tests(${test_name} EXTRA_ARGS --gtest_output=xml:${results_xml})
```
This creates the `test_my_stuff` target, links it to the gtest main target and
discovers all the gtest test cases and suites declared in the source code.
It also specifies the output location to be the `test_results` directory used
by Catkin if the test target is executed.

### Add the custom target to run the test
```cmake
    add_custom_target(
            _run_tests_${test_name}
            COMMAND ${test_name} --gtest_output=xml:${results_xml} || (exit 0))
    add_dependencies(_run_tests_${test_name} ${test_name})
```
The custom target executes the command to run test tests.
It has a dependency on the test target to ensure that it is built before
executing it.


### Ensure that the Catkin aggregate test targets exist
```cmake
    if(NOT TARGET tests)
        add_custom_target(tests)
    endif()
    if(NOT TARGET run_tests)
        add_custom_target(run_tests)
    endif()
```
Just in case they have not yet been created,  we check for the `tests` and
`run_tests` targets and create dummy custom targets if they don't exist.


### Add target dependencies to integrate with Catkin targets
```cmake
    add_dependencies(tests ${test_name})
    add_dependencies(run_tests _run_tests_${test_name})
endfunction()
```
Finally, the test target is added as a dependency to the `tests` target and the
`_run_test_${test_name}` target is added as a dependency to the `run_tests`
target to complete the integration with the Catkin test targets.

