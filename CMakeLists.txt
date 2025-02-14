cmake_minimum_required(VERSION 3.2 FATAL_ERROR)

project(surfelwarp)

# Add cmake module path
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

# Language options
set(CMAKE_C_COMPILER gcc-10)
set(CMAKE_CXX_COMPILER g++-10) #used?
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Cuda
find_package(CUDA REQUIRED)
include_directories(${CUDA_INCLUDE_DIRS})

add_definitions(-DCUB_USE_COOPERATIVE_GROUPS)

CUDA_SELECT_NVCC_ARCH_FLAGS(ARCH_FLAGS Auto)
LIST(APPEND CUDA_NVCC_FLAGS ${ARCH_FLAGS})
#set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -arch=sm_86")
#set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -gencode=arch=compute_86,code=sm_86")

# GLFW package
find_package(glfw3 REQUIRED)


# The library and compiler flags on Windows
if(MSVC)
    option( MSVC_USE_STATIC_CRT  "Use static C Runtime with MSVC, /MT instead of /MD" ON)

    # Make sure there are no erroneous C Runtime flags
    list(APPEND FLAG_VARS
        CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE CMAKE_CXX_FLAGS_MINSIZEREL CMAKE_CXX_FLAGS_RELWITHDEBINFO
        CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO
    )
    if(MSVC_USE_STATIC_CRT)
        foreach(FLAG_VAR ${FLAG_VARS})
            string(REGEX REPLACE "/MD" "/MT" NEW_FLAGS "${${FLAG_VAR}}")
            set(${FLAG_VAR} "${NEW_FLAGS}" CACHE STRING "" FORCE)
        endforeach()
    else()
        foreach(FLAG_VAR ${FLAG_VARS})
            string(REGEX REPLACE "/MT" "/MD" NEW_FLAGS "${${FLAG_VAR}}")
            set(${FLAG_VAR} "${NEW_FLAGS}" CACHE STRING "" FORCE)
        endforeach()
    endif()

	# Provisions for typical Boost compiled on Windows
    # Most commonly, Boost libraries are compiled statically on windows (change as necesssary)
    set(Boost_USE_STATIC_LIBS TRUE)
	if(MSVC_USE_STATIC_CRT)
		set(Boost_USE_STATIC_RUNTIME ON)
	else()
		set(Boost_USE_STATIC_RUNTIME OFF)
	endif()
	set(Boost_USE_MULTITHREADED ON)
	set(Boost_USE_DEBUG_RUNTIME ON)
	set(Boost_USE_DEBUG_PYTHON OFF)
endif(MSVC)
# The library and compiler flags on Ubuntu
if(UNIX)
    #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")
    set(CUDA_NVCC_FLAGS "${CUDA_NVCC_FLAGS} -g -G")
endif(UNIX)

# Append "d" to library versions with debugging symbols
set(CMAKE_DEBUG_POSTFIX d)

# Do not use the system Eigen
set(EIGEN_ROOT "${CMAKE_SOURCE_DIR}/external/eigen3.4")
set(EIGEN_INCLUDE_DIR ${EIGEN_ROOT})

# Point cloud library and a series / cilantro


set(VISUALIZATION_LIBRARY "PCL" CACHE STRING "Which library to use for visualization")
set_property(CACHE VISUALIZATION_LIBRARY PROPERTY STRINGS "PCL" "cilantro")
if(VISUALIZATION_LIBRARY STREQUAL "PCL")
    find_package(PCL REQUIRED)
    include_directories(${PCL_INCLUDE_DIRS})
    add_definitions(-DWITH_PCL)
elseif(VISUALIZATION_LIBRARY STREQUAL "cilantro")
    find_package(cilantro REQUIRED)
    include_directories(${cilantro_INCLUDE_DIRS})
    add_definitions(-DWITH_CILANTRO)
endif()


# Check the CUDA version and resolve confliction with pcl
if(${CUDA_VERSION_MAJOR} LESS 11)
    if(VISUALIZATION_LIBRARY STREQUAL "PCL")
        list(REMOVE_ITEM PCL_LIBRARIES "vtkproj4")
    endif()
    get_directory_property(dir_defs DIRECTORY ${CMAKE_SOURCE_DIR} COMPILE_DEFINITIONS)
    set(vtk_flags)
    foreach(it ${dir_defs})
        if(it MATCHES "vtk*")
            list(APPEND vtk_flags ${it})
        endif()
    endforeach()

    foreach(d ${vtk_flags})
        remove_definitions(-D${d})
    endforeach()
endif()

# OpenCV lib: this version uses the customized build
find_package(OpenCV REQUIRED)
include_directories(${OpenCV_INCLUDE_DIR})

# The boost library is contained in pcl
find_package(Boost REQUIRED COMPONENTS system filesystem)
include_directories(${Boost_INCLUDE_DIRS})

# The project directory shall also be included
include_directories("${CMAKE_SOURCE_DIR}")
include_directories("${CMAKE_SOURCE_DIR}/external")

# The external library
add_subdirectory(external)

# The libraries
add_subdirectory(common)
add_subdirectory(math)
add_subdirectory(pcg_solver)
add_subdirectory(imgproc)
add_subdirectory(visualization)
add_subdirectory(core)


# The hashing libaray should not depends on PCL
add_subdirectory(hashing)

# Applications
add_subdirectory(apps)
