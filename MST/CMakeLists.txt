################
# Iman Jehanzeb
################

cmake_minimum_required(VERSION 3.22)

cmake_host_system_information(RESULT num QUERY NUMBER_OF_PHYSICAL_CORES)

cmake_host_system_information(RESULT name QUERY DISTRIB_PRETTY_NAME)

message(STATUS "Running CMake ${CMAKE_VERSION} for MST with max ${num} threads on ${name}")

project(MST LANGUAGES Fortran C)


# make CUDA optional
include(CheckLanguage)
check_language(CUDA)
if(CMAKE_CUDA_COMPILER)
        enable_language(CUDA)
	message("CUDA compiler found, using CUDA")
else()
        message(STATUS "No CUDA compiler found")
endif()




# Defining mst_main
add_executable(mst_main)

set_target_properties(mst_main PROPERTIES OUTPUT_NAME "MST")

set_target_properties(mst_main PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")


# Defining mst_lib 
add_library(mst_lib)



set_target_properties(mst_lib PROPERTIES
        LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
        ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
        Fortran_MODULE_DIRECTORY ${CMAKE_BINARY_DIR}/modules)
        
target_include_directories(mst_lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

target_link_libraries(mst_main PUBLIC mst_lib)
target_link_libraries(mst_main PUBLIC gfortran)


# Adding Subdirectories
add_subdirectory(src)
add_subdirectory(lib)
add_subdirectory(plib)
add_subdirectory(iolib)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)




#####################
# Find OpenMP library
#####################
option(BUILD_WITH_OPENMP "Build with OpenMP" ON)
if (BUILD_WITH_OPENMP)
    find_package(OpenMP REQUIRED)
    target_link_libraries(mst_lib PUBLIC OpenMP::OpenMP_Fortran)
    message(STATUS "OpenMP found: " "${OpenMP_Fortran_FOUND}")
    message("Using OpenMP (Fortran)")
endif()



#############      
# Build LibXC 
#############
option(USE_LIBXC "LibXC" ON)
if (USE_LIBXC)
	include(ExternalProject)
	ExternalProject_Add(libxc
		    GIT_REPOSITORY https://github.com/ElectronicStructureLibrary/			    libxc.git)
	message("Using LibXC")

endif()




###################
# Find MPI
####################
option(BUILD_WITH_MPI "Build with MPI" ON)
if (BUILD_WITH_MPI)
	
	find_package(MPI REQUIRED)
	include_directories(${MPI_INCLUDE_PATH})
	target_link_libraries(mst_lib PUBLIC MPI::MPI_Fortran)
	message("Using MPI")
endif()





############
# CUDA
############
option(USE_ACCELERATOR_CUDA_C OFF)
if (USE_ACCELERATOR_CUDA_C)

    target_compile_definitions(mst_lib PUBLIC ACCELERATOR_CUDA_C)

    option(CUDA_DEBUG "Debug CUDA routines" ON)

    if (CUDA_DEBUG)
        target_compile_definitions(mst_lib PUBLIC "CUDA_DEBUG")
endif ()



if (NOT DEFINED CMAKE_CUDA_STANDARD)
        set(CMAKE_CUDA_STANDARD 14)
        set(CMAKE_CUDA_STANDARD_REQUIRED ON)
endif ()

if (NOT DEFINED CMAKE_CUDA_SEPARABLE_COMPILATION)
        set(CMAKE_CUDA_SEPARABLE_COMPILATION OFF)
endif ()

find_package(CUDAToolkit OPTIONAL)


message(STATUS "CUDA Version: " ${CUDAToolkit_VERSION})
message(STATUS "CUDA Include dirs: " ${CUDAToolkit_INCLUDE_DIRS})
message(STATUS "CUDA Library: " ${CUDAToolkit_LIBRARY_DIR})


target_link_libraries(mst_lib PUBLIC CUDA::toolkit)

# Check if the necessary libraries have been found
if (TARGET CUDA::cublas)
        target_link_libraries(mst_lib PUBLIC CUDA::cublas)
else ()
        message(FATAL_ERROR "CuBLAS was not defined or found")
endif ()

if (TARGET CUDA::cusolver)
        target_link_libraries(mst_lib PUBLIC CUDA::cusolver)
else ()
        message(FATAL_ERROR "Cusolver was not defined or found")
    endif ()
endif ()




############
# Find HDF5
############
option(USE_HDF5 "Use HDF5" OFF)
if(USE_HDF5)
	message(STATUS "Using HDF5")
	find_package(HDF5 REQUIRED)

	message(STATUS "HDF5 Version:" "${HDF5_VERSION}")

	target_link_libraries(mst_lib PUBLIC HDF5::HDF5)
endif()

###########################
# Linear algebra libraries
###########################
option(USE_ESSL "Use IBM ESSL library" OFF)

if (USE_ESSL)
    find_package(ESSL REQUIRED)
    set(ESSL_WORKAROUND TRUE)
    target_link_libraries(mst_lib PUBLIC ESSL::ESSL)
    message("Using ESSL")
else ()

    # Lapack

    option(SEARCH_LAPACK "Search for LAPACK" ON)
    if (NOT SEARCH_LAPACK)
        add_library(LAPACK::LAPACK STATIC IMPORTED GLOBAL)
        set_target_properties(LAPACK::LAPACK PROPERTIES IMPORTED_LOCATION ${LAPACK_LIBRARIES})
        if (DEFINED LAPACK_LINKER_FLAGS)
            target_compile_options(LAPACK::LAPACK INTERFACE ${LAPACK_LINKER_FLAGS})
        endif ()
        if (DEFINED LAPACK_INCLUDE_DIR)
            target_include_directories(LAPACK::LAPACK INTERFACE ${LAPACK_INCLUDE_DIR})
        endif ()
    else ()
        if (NOT DEFINED BLA_VENDOR)
            set(BLA_VENDOR "Generic")
        endif ()
        find_package(LAPACK REQUIRED)
	message("Using Lapack")
    endif ()
    target_link_libraries(mst_lib PUBLIC LAPACK::LAPACK)



##############
# BLAS
##############

option(SEARCH_BLAS "Search for BLAS" ON)

if (NOT SEARCH_BLAS)
        add_library(BLAS::BLAS STATIC IMPORTED GLOBAL)
        set_target_properties(BLAS::BLAS PROPERTIES IMPORTED_LOCATION ${BLAS_LIBRARIES})
        if (DEFINED BLAS_LINKER_FLAGS)
                target_compile_options(BLAS::BLAS INTERFACE ${BLAS_LINKER_FLAGS})
        endif ()
        if (DEFINED BLAS_INCLUDE_DIR)
                target_include_directories(BLAS::BLAS INTERFACE ${BLAS_INCLUDE_DIR})
        endif ()

        else ()
                if (NOT DEFINED BLA_VENDOR)
                        set(BLA_VENDOR "Generic")
                endif ()
                find_package(BLAS REQUIRED)
		message("Using BLAS")

        endif ()
        target_link_libraries(mst_lib PUBLIC BLAS::BLAS)
endif ()
