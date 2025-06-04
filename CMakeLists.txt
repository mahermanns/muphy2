cmake_minimum_required(VERSION 3.29)

# Check for git executable
find_package(Git)
if (GIT_EXECUTABLE)
    # Check if in compiled from Git repository
    execute_process(
        COMMAND ${GIT_EXECUTABLE} rev-parse --is-inside-work-tree
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_IS_REPO
        OUTPUT_STRIP_TRAILING_WHITESPACE
        RESULT_VARIABLE GIT_RESULT
    )

    if(GIT_RESULT EQUAL 0 AND GIT_IS_REPO STREQUAL "true")
        # Get a version string from git
        execute_process(COMMAND ${GIT_EXECUTABLE} describe --always --tags --dirty=* --abbrev=8
                        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                        OUTPUT_VARIABLE MUPHY2_GIT_VERSION)
        string(STRIP "${MUPHY2_GIT_VERSION}" MUPHY2_GIT_VERSION)
        set(MUPHY2_VERSION git-${MUPHY2_GIT_VERSION} CACHE STRING "Version of MUPHY2")
    else()
        set(MUPHY2_VERSION rel-1.0.1 CACHE STRING "Version of MUPHY2")
    endif()
    message(STATUS "MUPHY2 version: ${MUPHY2_VERSION}")
endif()

# Create Project and enable CXX and Fortran compilers
project(muphy2
    LANGUAGES CXX Fortran)

# Detect and set correct CXX flags for C++11
set(CMAKE_CXX_STANDARD 11)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING "Choose the type of build." FORCE)
endif()

# Set precision of floating point operations
set(MUPHY2_PRECISION double CACHE STRING "Select precision: 'single' or 'double'")
set_property(CACHE MUPHY2_PRECISION PROPERTY STRINGS single double)
message(STATUS "Selected precision: ${MUPHY2_PRECISION}")
if("${MUPHY2_PRECISION}" STREQUAL "single")
    target_compile_options(muphy2 PRIVATE -DSINGLE_PRECISION)
endif()

# Create an include file with CMake variables
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/framework/version.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/version.h
    @ONLY
)

# Create target for 'muphy2' executable
add_executable(muphy2
    ${CMAKE_CURRENT_BINARY_DIR}/version.h)

# Add MPI depenency
target_link_libraries(muphy2 PRIVATE MPI::MPI_CXX MPI::MPI_Fortran)
# Check for MPI support
find_package(MPI REQUIRED)

# Enable check for OpenACC if NVHPC is used, otherwise disable automatic check
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "NVHPC")
    set(OPENACC_DEFAULT ON)
elseif()
    set(OPENACC_DEFAULT OFF)
endif()

option(ENABLE_OPENACC "Enable OpenACC support" ${OPENACC_DEFAULT})
option(ENABLE_CUDA "Enable CUDA support" ${OPENACC_DEFAULT})
if (ENABLE_OPENACC)
    # Check for OpenACC support
    find_package(OpenACC)
    if (OpenACC_FOUND)
        message(STATUS "Enabling OpenACC support")
        target_link_libraries(muphy2 PRIVATE OpenACC::OpenACC_CXX OpenACC::OpenACC_Fortran)
        target_compile_options(muphy2 PRIVATE
            $<$<COMPILE_LANGUAGE:CXX>:${OpenACC_CXX_OPTIONS}>
            $<$<COMPILE_LANGUAGE:Fortran>:${OpenACC_Fortran_OPTIONS}>
        )

        # Set 2D vs 3D optimizations for OpenACC pragmas
        set(ACC_OPT "3D" CACHE STRING "Select dimensionality to optimize ACC pragmas for: 2D or 3D")
        set_property(CACHE ACC_OPT PROPERTY STRINGS 2D 3D)
        message(STATUS "Optimizing ACC pragmas for: ${ACC_OPT}")
        if("${ACC_OPT}" STREQUAL "2D")
            target_compile_options(muphy2 PRIVATE -DACC_2D_OPT)
        endif()

        if (ENABLE_CUDA)
            # Check for CUDA, will fail configure if not found
            enable_language(CUDA)
            if (CUDA_FOUND)
                message(STATUS "Enabling CUDA support")
            endif()
        endif()
    endif()
else()
    message(STATUS "Disabling OpenACC support")
endif()

# Add specific initial conditions file to target dependencies
set(MUPHY2_INITIAL_CONDITIONS_FILE "${CMAKE_CURRENT_SOURCE_DIR}/framework/initial_conditions.cc" CACHE FILEPATH "Source file for initial conditions")
if (IS_READABLE ${MUPHY2_INITIAL_CONDITIONS_FILE})
    message(STATUS "Using initial conditions file: ${MUPHY2_INITIAL_CONDITIONS_FILE}")
else()
    message(STATUS "Specified initial conditions file '${MUPHY2_INITIAL_CONDITIONS_FILE}' does not exist or is not readable.")
endif()

# Generate framework/initial_conditions.cc
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/framework/initial_conditions.cc
    COMMAND cp ${MUPHY2_INITIAL_CONDITIONS_FILE} ${CMAKE_CURRENT_BINARY_DIR}/framework/initial_conditions.cc
    DEPENDS ${MUPHY2_INITIAL_CONDITIONS_FILE}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    COMMENT "Generating 'framework/initial_conditions.cc'"
)
target_sources(muphy2 PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/framework/initial_conditions.cc)

# Generate Hex-dumped headers
set(MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT initial_conditions_files_content.h)
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT}
    COMMAND xxd -i ${CMAKE_CURRENT_BINARY_DIR}/framework/initial_conditions.cc | sed -e "s/_.*_initial_conditions_cc/___framework_initial_conditions_cc/" > ${CMAKE_CURRENT_BINARY_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT}
    COMMAND xxd -i framework/parameter.h | sed -e "s/[ ]*framework_parameter_h/ ___framework_parameter_h/" >> ${CMAKE_CURRENT_BINARY_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT}
    COMMAND xxd -i physics/plasma/setup/setup.F90 | sed -e "s/physics.*setup_F90/___physics_plasma_setup_setup_F90/" >> ${CMAKE_CURRENT_BINARY_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT}
    DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/framework/initial_conditions.cc framework/parameter.h physics/plasma/setup/setup.F90
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    COMMENT "Generating '${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT}'"
    )
# Add generated file to target source dependencies
target_sources(muphy2 PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT})

# Add base directory to include path (to satisfy '#include "framework/..."' statements)
target_include_directories(muphy2 PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})
# Add build directory to include path (to enable finding generated sources)
target_include_directories(muphy2 PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

add_subdirectory(framework)
add_subdirectory(physics/plasma/converters)
add_subdirectory(physics/plasma/criteria)
add_subdirectory(physics/plasma/models)
add_subdirectory(physics/plasma/output)
add_subdirectory(physics/plasma/schemes)
add_subdirectory(physics/plasma/setup)

# Assemble compile time options for target
target_compile_options(muphy2 PRIVATE
    $<$<CXX_COMPILER_ID:NVHPC>:-fast>
    $<$<CXX_COMPILER_ID:Clang,GNU,Intel,IntelLLVM>:-Ofast -march=native -mtune=native>
    $<$<COMPILE_LANGUAGE:CXX>:$<$<CXX_COMPILER_ID:Clang,GNU,Intel,IntelLLVM>:-Wno-unknown-pragmas>>
    $<$<CXX_COMPILER_ID:GNU>:-MMD>
    $<$<COMPILE_LANGUAGE:Fortran>:$<$<Fortran_COMPILER_ID:GNU>:-fPIE -ffree-line-length-none>>
)
target_link_libraries(muphy2 PRIVATE
    $<$<CXX_COMPILER_ID:GNU>:-lgfortran>
)

install(TARGETS muphy2 DESTINATION bin)
