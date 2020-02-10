project(nlohmann_json_schema_validator
    LANGUAGES CXX)

set(PROJECT_VERSION 2.1.0)

cmake_minimum_required(VERSION 3.2)

option(BUILD_TESTS      "Build tests"    ON)
option(BUILD_EXAMPLES   "Build examples" ON)


find_package(nlohmann_json REQUIRED)
get_target_property(nlohmann_json_INCLUDES nlohmann_json::nlohmann_json INTERFACE_INCLUDE_DIRECTORIES)
include_directories(
  ${nlohmann_json_INCLUDES} # For nlohmann/json_fwd.hpp present in bci header files
)

# the library
add_library(nlohmann_json_schema_validator
    src/json-schema-draft7.json.cpp
    src/json-uri.cpp
    src/json-validator.cpp
    src/string-format-check.cpp)


set_target_properties(${PROJECT_NAME} PROPERTIES 
  DEBUG_POSTFIX "d"
  MINSIZEREL_POSTFIX "msr"
  RELWITHDEBINFO_POSTFIX "rwdi")


target_include_directories(nlohmann_json_schema_validator
    PUBLIC
        $<INSTALL_INTERFACE:include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>)

target_include_directories(nlohmann_json_schema_validator
    PUBLIC
        $<BUILD_INTERFACE:${nlohmann_json_INCLUDES}>)

target_compile_features(nlohmann_json_schema_validator
    PUBLIC
        cxx_range_for) # for C++11 - flags

target_link_libraries(${PROJECT_NAME} PRIVATE nlohmann_json::nlohmann_json)

set_target_properties(nlohmann_json_schema_validator
                      PROPERTIES
                          VERSION ${PROJECT_VERSION}
                          SOVERSION 1)

# if used as a sub-directory, do not create install-rules -
# because of the dependency to nlohmann_json.
set(JSON_VALIDATOR_INSTALL ON)



if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" OR
   "${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    target_compile_options(nlohmann_json_schema_validator
        PRIVATE
            -Wall -Wextra)
endif()

if(BUILD_SHARED_LIBS)
    target_compile_definitions(nlohmann_json_schema_validator
        PRIVATE
            -DJSON_SCHEMA_VALIDATOR_EXPORTS)
endif()

# regex with boost if gcc < 4.9 - default is std::regex
if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "4.9.0")
        find_package(Boost COMPONENTS regex)
        if(NOT Boost_FOUND)
            message(STATUS "GCC less then 4.9 and boost-regex NOT found - no regex used")
            target_compile_definitions(nlohmann_json_schema_validator PRIVATE -DJSON_SCHEMA_NO_REGEX)
        else()
            message(STATUS "GCC less then 4.9 and boost-regex FOUND - using boost::regex")
            target_compile_definitions(nlohmann_json_schema_validator PRIVATE -DJSON_SCHEMA_BOOST_REGEX)
            target_include_directories(nlohmann_json_schema_validator PRIVATE ${Boost_INCLUDE_DIRS})
            target_link_libraries(nlohmann_json_schema_validator PRIVATE ${Boost_LIBRARIES})
        endif()
    endif()
endif()

if(JSON_VALIDATOR_INSTALL)
    install(TARGETS nlohmann_json_schema_validator
            EXPORT ${PROJECT_NAME}Targets
            LIBRARY DESTINATION lib
            ARCHIVE DESTINATION lib
            RUNTIME DESTINATION bin)

    install(FILES src/nlohmann/json-schema.hpp
            DESTINATION include/nlohmann)
endif()

if (BUILD_EXAMPLES)
    # simple nlohmann_json_schema_validator-executable
    add_executable(json-schema-validate app/json-schema-validate.cpp)
    target_link_libraries(json-schema-validate nlohmann_json_schema_validator)

    add_executable(readme-json-schema app/readme.cpp)
    target_link_libraries(readme-json-schema nlohmann_json_schema_validator)

    install(TARGETS json-schema-validate readme-json-schema
            DESTINATION bin)
endif()

if (BUILD_TESTS)
    # test-zone
    enable_testing()
    add_subdirectory(test)
endif()

if(JSON_VALIDATOR_INSTALL)
    # Set Up the Project Targets and Config Files for CMake

    # Set the install path to the cmake config files
    set(INSTALL_CMAKE_DIR ${CMAKE_INSTALL_PREFIX}/lib/cmake/${PROJECT_NAME})

    # Create the ConfigVersion file
    include(CMakePackageConfigHelpers) # write_basic_package_version_file
    write_basic_package_version_file( ${PROJECT_NAME}ConfigVersion.cmake
                                      VERSION ${PACKAGE_VERSION}
                                      COMPATIBILITY SameMajorVersion)

    # Get the relative path from the INSTALL_CMAKE_DIR to the include directory
    file(RELATIVE_PATH REL_INCLUDE_DIR "${INSTALL_CMAKE_DIR}" "${CMAKE_INSTALL_PREFIX}/include")


    # Configure the Config.cmake file with the proper include directory
    set(CONF_INCLUDE_DIRS "\${JSON_SCHEMA_VALIDATOR_CMAKE_DIR}/${REL_INCLUDE_DIR}")
    configure_file(${PROJECT_NAME}Config.cmake.in
                   "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake" @ONLY)

    # Install the Config.cmake and ConfigVersion.cmake files
    install(FILES
            "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
            DESTINATION "${INSTALL_CMAKE_DIR}")

    # Install Targets
    install(EXPORT ${PROJECT_NAME}Targets
            FILE ${PROJECT_NAME}Targets.cmake
            DESTINATION "${INSTALL_CMAKE_DIR}")
endif()
