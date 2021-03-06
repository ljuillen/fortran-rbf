cmake_minimum_required(VERSION 2.8)

project(rbf LANGUAGES Fortran)

if (CMAKE_Fortran_COMPILER_ID STREQUAL "GNU")
  set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -ffpe-summary=none -ffree-line-length-none -Ofast -fopenmp")
endif ()

if (DEFINED ENV{LAPACK_ROOT})
  set(USE_LAPACK ON)
  add_definitions(-DUSE_LAPACK)
  link_libraries("$ENV{LAPACK_ROOT}/lib")
  set(LAPACK_LIBRARIES "lapack")
else ()
  if (APPLE)
    set(BLA_VENDOR "Apple")
  endif ()
  find_package(LAPACK)
  if (LAPACK_FOUND)
    set(USE_LAPACK ON)
    add_definitions(-DUSE_LAPACK)
  else ()
    set(USE_LAPACK OFF)
  endif ()
endif ()

if (DEFINED ENV{GSL} AND (NOT DEFINED ENV{GSL_ROOT}))
  set(ENV{GSL_ROOT} $ENV{GSL})
endif ()
if (NOT USE_LAPACK AND DEFINED ENV{GSL_ROOT})
  set(USE_GSL ON)
  add_definitions(-DUSE_GSL)
  include_directories("$ENV{GSL_ROOT}/include")
  link_directories("$ENV{GSL_ROOT}/lib")
endif ()

if (DEFINED ENV{NETCDF} AND (NOT DEFINED ENV{NETCDF_ROOT}))
  set(ENV{NETCDF_ROOT} $ENV{NETCDF})
endif ()
if (DEFINED ENV{NETCDF_ROOT})
  include_directories("$ENV{NETCDF_ROOT}/include")
  link_directories("$ENV{NETCDF_ROOT}/lib")
else ()
  find_package(PkgConfig)
  if (PKG_CONFIG_FOUND)
    pkg_search_module(NETCDF REQUIRED netcdf)
    if (NETCDF_FOUND)
      include_directories(${NETCDF_INCLUDEDIR})
      link_directories(${NETCDF_LIBDIR})
    else ()
      message(FATAL_ERROR "Pkg-config could not find netcdf library!")
    endif ()
  else ()
    message(FATAL_ERROR "Unable to find pkg-config library!")
  endif ()
endif ()

add_subdirectory(lib/container)
add_subdirectory(lib/flogger)
add_subdirectory(lib/fiona)
add_subdirectory(lib/kdtree)

set(sources
  src/const_mod.F90
  src/distance_mod.F90
  src/math_mod.F90
  src/kernel_mod.F90
  src/rbf_mod.F90
)

add_library(rbf ${sources})
if (USE_LAPACK)
  target_link_libraries(rbf ${LAPACK_LIBRARIES})
endif ()
if (USE_GSL)
  target_link_libraries(rbf gsl)
endif ()

set(sources
  src/resize_array_mod.F90
  src/node_placing_mod.F90
  src/prepare_nodes.F90
)

add_executable(prepare_nodes.exe ${sources})
target_link_libraries(prepare_nodes.exe flogger fiona kdtree)

set(sources
  src/rbf_test.F90
)

add_executable(rbf_test.exe ${sources})
target_link_libraries(rbf_test.exe fiona rbf)
