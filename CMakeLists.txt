cmake_minimum_required(VERSION 3.16)

# Check for git executable
find_package(Git)
if (GIT_EXECUTABLE)
    # Get a version string from git
    execute_process(COMMAND ${GIT_EXECUTABLE} describe --all --contains --tags --dirty="*" --abbrev=8
                    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                    OUTPUT_VARIABLE MUPHY2_GIT_VERSION)
    string(STRIP "${MUPHY2_GIT_VERSION}" MUPHY2_GIT_VERSION)
    message(STATUS "Detected Git version: ${MUPHY2_GIT_VERSION}")
endif()

# Create Project and enable CXX and Fortran compilers
project(muphy2
    LANGUAGES CXX Fortran)

# Detect and set correct CXX flags for C++11
set(CMAKE_CXX_STANDARD 11)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING "Choose the type of build." FORCE)
endif()

# Create target for 'muphy2' executable
add_executable(muphy2)

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
set(MUPHY2_INITIAL_CONDITIONS_DIR ${CMAKE_CURRENT_SOURCE_DIR}/physics/plasma/setup/initial_conditions CACHE PATH "Source directory for initial conditions file")
set(MUPHY2_INITIAL_CONDITIONS_FILE initial_conditions_ot.cc CACHE FILEPATH "Source file for initial conditions")
message(STATUS "Using initial conditions file: ${MUPHY2_INITIAL_CONDITIONS_FILE}")
target_sources(muphy2 PRIVATE ${MUPHY2_INITIAL_CONDITIONS_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILE})

# Generate Hex-dumped headers
set(MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT initial_conditions_files_content.h)
add_custom_command(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT}
    COMMAND cp ${MUPHY2_INITIAL_CONDITIONS_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILE} ${CMAKE_CURRENT_BINARY_DIR}/framework/initial_conditions.cc
    COMMAND xxd -i ${CMAKE_CURRENT_BINARY_DIR}/framework/initial_conditions.cc | sed -e "s/_.*_initial_conditions_cc/___framework_initial_conditions_cc/" > ${CMAKE_CURRENT_BINARY_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT}
    COMMAND xxd -i framework/parameter.h | sed -e "s/[ ]*framework_parameter_h/ ___framework_parameter_h/" >> ${CMAKE_CURRENT_BINARY_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT}
    COMMAND xxd -i physics/plasma/setup/setup.F90 | sed -e "s/physics.*setup_F90/___physics_plasma_setup_setup_F90/" >> ${CMAKE_CURRENT_BINARY_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT}
    DEPENDS ${MUPHY2_INITIAL_CONDITIONS_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILE} framework/parameter.h physics/plasma/setup/setup.F90
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    COMMENT "Generating '${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT}'"
    )
target_sources(muphy2 PRIVATE ${CMAKE_CURRENT_BINARY_DIR}/${MUPHY2_INITIAL_CONDITIONS_FILES_CONTENT})

target_include_directories(muphy2 PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
target_include_directories(muphy2 PRIVATE ${CMAKE_CURRENT_SOURCE_DIR})

add_subdirectory(framework)
add_subdirectory(physics/plasma/converters)
add_subdirectory(physics/plasma/criteria)
add_subdirectory(physics/plasma/models)
add_subdirectory(physics/plasma/output)
add_subdirectory(physics/plasma/schemes)
add_subdirectory(physics/plasma/setup)

target_compile_options(muphy2 PRIVATE
    $<$<CXX_COMPILER_ID:NVHPC>:-fast>
    $<$<CXX_COMPILER_ID:GNU,Intel,IntelLLVM>:-Ofast -march=native -mtune=native>
    $<$<CXX_COMPILER_ID:GNU>:-lgfortran -MMD>
    $<$<COMPILE_LANGUAGE:Fortran>:$<$<Fortran_COMPILER_ID:GNU>:-ffree-line-length-none>>
)

install(TARGETS muphy2 DESTINATION bin)
