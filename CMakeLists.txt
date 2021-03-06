# CMake powered build system for OpenMVS

########################################################################
#
# Project-wide settings
CMAKE_MINIMUM_REQUIRED(VERSION 2.6.2)

# Find dependencies:
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/build/Modules)

# Name of the project.
#
# CMake files in this project can refer to the root source directory
# as ${OpenMVS_SOURCE_DIR} and to the root binary directory as
# ${OpenMVS_BINARY_DIR}.
PROJECT(OpenMVS)

# fix CMake IntDir variable
if(MSVC AND "${MSVC_VERSION}" STRGREATER "1500")
	SET(CMAKE_CFG_INTDIR "$(Platform)/$(Configuration)")
endif()
SET(COTIRE_INTDIR "cotire")

# Define helper functions and macros.
cmake_policy(SET CMP0011 OLD)
INCLUDE(build/Utils.cmake)
if(ENABLE_PRECOMPILED_HEADERS)
	INCLUDE(build/Cotire.cmake)
endif()


# Init session with macros defined in Utils.cmake
GetOperatingSystemArchitectureBitness(SYSTEM)
ComposePackageLibSuffix()
ConfigCompilerAndLinker()

# List configuration options
SET(OpenMVS_USE_NONFREE ON CACHE BOOL "Build non-free (patented) functionality")
SET(OpenMVS_USE_EXIV2 OFF CACHE BOOL "Link and use EXIV2 library")
SET(OpenMVS_USE_FAST_FLOAT2INT ON CACHE BOOL "Use an optimized code to convert real numbers to int")
SET(OpenMVS_USE_FAST_INVSQRT OFF CACHE BOOL "Use an optimized code to compute the inverse square root (slower in fact on modern compilers)")
SET(OpenMVS_USE_FAST_CBRT ON CACHE BOOL "Use an optimized code to compute the cubic root")
SET(OpenMVS_USE_SSE ON CACHE BOOL "Enable SSE optimizations")
SET(OpenMVS_USE_OPENMP ON CACHE BOOL "Enable OpenMP library")
SET(OpenMVS_USE_CUDA ON CACHE BOOL "Enable CUDA library")
SET(OpenMVS_USE_BREAKPAD ON CACHE BOOL "Enable BreakPad library")
SET(OpenMVS_CONFIG_INCLUDE_DIR "${CMAKE_BINARY_DIR}/" CACHE PATH "Where to create the build/platform specific header")

INCLUDE_DIRECTORIES("${OpenMVS_SOURCE_DIR}")

# Find required packages
if(OpenMVS_USE_OPENMP)
	FIND_PACKAGE(OpenMP)
	if(OPENMP_FOUND)
		SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
		ADD_DEFINITIONS(-D_USE_OPENMP)
		SET(_USE_OPENMP TRUE)
	else()
		message("-- Can't find OpenMP. Continuing without it.")
	endif()
endif()

if(OpenMVS_USE_CUDA)
	FIND_PACKAGE(CUDA)
	if(CUDA_FOUND)
		INCLUDE_DIRECTORIES(${CUDA_INCLUDE_DIRS})
		ADD_DEFINITIONS(-D_USE_CUDA)
		SET(_USE_CUDA TRUE)
	else()
		SET(CUDA_LIBRARIES "")
		MESSAGE("-- Can't find CUDA. Continuing without it.")
	endif()
endif()

if(OpenMVS_USE_BREAKPAD)
	FIND_PACKAGE(BREAKPAD)
	if(BREAKPAD_FOUND)
		INCLUDE_DIRECTORIES(${BREAKPAD_INCLUDE_DIRS})
		ADD_DEFINITIONS(${BREAKPAD_DEFINITIONS} -D_USE_BREAKPAD)
		SET(_USE_BREAKPAD TRUE)
	else()
		MESSAGE("-- Can't find BreakPad. Continuing without it.")
	endif()
endif()

FIND_PACKAGE(Boost ${SYSTEM_PACKAGE_REQUIRED} COMPONENTS iostreams program_options system serialization)
if(Boost_FOUND)
	INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIRS})
	ADD_DEFINITIONS(${Boost_DEFINITIONS} -D_USE_BOOST)
	LINK_DIRECTORIES(${Boost_LIBRARY_DIRS})
	SET(_USE_BOOST TRUE)
endif()

FIND_PACKAGE(Eigen ${SYSTEM_PACKAGE_REQUIRED})
if(EIGEN_FOUND)
	INCLUDE_DIRECTORIES(${EIGEN_INCLUDE_DIRS})
	ADD_DEFINITIONS(${EIGEN_DEFINITIONS} -D_USE_EIGEN)
	SET(_USE_EIGEN TRUE)
endif()

FIND_PACKAGE(OpenCV ${SYSTEM_PACKAGE_REQUIRED})
if(OpenCV_FOUND)
	INCLUDE_DIRECTORIES(${OpenCV_INCLUDE_DIRS})
	ADD_DEFINITIONS(${OpenCV_DEFINITIONS})
	SET(_USE_OPENCV TRUE)
endif()

# Set defines
SET(OpenMVS_DEFINITIONS "")
if(OpenMVS_USE_NONFREE)
	LIST(APPEND OpenMVS_DEFINITIONS -D_USE_NONFREE)
	SET(_USE_NONFREE TRUE)
endif()
if(OpenMVS_USE_FAST_FLOAT2INT)
	LIST(APPEND OpenMVS_DEFINITIONS -D_USE_FAST_FLOAT2INT)
	SET(_USE_FAST_FLOAT2INT TRUE)
endif()
if(OpenMVS_USE_FAST_INVSQRT)
	LIST(APPEND OpenMVS_DEFINITIONS -D_USE_FAST_INVSQRT)
	SET(_USE_FAST_INVSQRT TRUE)
endif()
if(OpenMVS_USE_FAST_CBRT)
	LIST(APPEND OpenMVS_DEFINITIONS -D_USE_FAST_CBRT)
	SET(_USE_FAST_CBRT TRUE)
endif()
if(OpenMVS_USE_SSE)
	LIST(APPEND OpenMVS_DEFINITIONS -D_USE_SSE)
	SET(_USE_SSE TRUE)
endif()
ADD_DEFINITIONS(${OpenMVS_DEFINITIONS})

# Add modules
ADD_SUBDIRECTORY(libs)
ADD_SUBDIRECTORY(apps)
ADD_SUBDIRECTORY(docs)

# Set configuration file
CONFIGURE_FILE("${OpenMVS_SOURCE_DIR}/build/Templates/ConfigLocal.h.in" "${OpenMVS_CONFIG_INCLUDE_DIR}/ConfigLocal.h")
INSTALL(FILES "${OpenMVS_CONFIG_INCLUDE_DIR}/ConfigLocal.h" DESTINATION "include/${PROJECT_NAME}")
INCLUDE_DIRECTORIES(${OpenMVS_CONFIG_INCLUDE_DIR})
