cmake_minimum_required(VERSION 2.6)

project (HASH)
# The version number.
set (HASH_VERSION_MAJOR 2)
set (HASH_VERSION_MINOR 0.7)
set (HASH_VERSION "${HASH_VERSION_MAJOR}.${HASH_VERSION_MINOR}")

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake/Modules/")
# --- custom targets: ---
if (NOT TARGET distclean)
   INCLUDE( cmake/Modules/TargetDistclean.cmake OPTIONAL)
endif (NOT TARGET distclean)

# enable @rpath in the install name for any shared library being built
set(CMAKE_MACOSX_RPATH 1)

if (NOT CMAKE_BUILD_TYPE)
   set(CMAKE_BUILD_TYPE RelWithDebInfo)
endif(NOT CMAKE_BUILD_TYPE)

# uncomment to build debug
#set(CMAKE_BUILD_TYPE Debug)

set (CMAKE_C_STANDARD 99)

if ("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang")
   # using Clang
   set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ftree-vectorize")
   #set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ftree-vectorize -ftree-vectorizer-verbose=5")
   set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -Wall -g -O0 -Wextra")
   set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -g -O3")
elseif ("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")
   # using GCC
   set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -Wall -g -O0 -Wextra")
   set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -g -O3 -fstrict-aliasing -ftree-vectorize")
   set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -fopt-info-vec-optimized")
   if ("${CMAKE_C_COMPILER_VERSION}" VERSION_GREATER "4.9")
     set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -fopenmp-simd")
   endif ("${CMAKE_C_COMPILER_VERSION}" VERSION_GREATER "4.9")
   if ("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86_64")
     set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -march=native -mtune=native")
   elseif ("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "ppc64le")
     set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -mcpu=powerpc64le")
   else ()
     set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
   endif ()
elseif ("${CMAKE_C_COMPILER_ID}" STREQUAL "Intel")
   # using Intel C
   set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -Wall -g -O0 -Wextra")
   set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -g -O3 -ansi-alias -vecabi=cmdtarget -xHost -qopenmp-simd -qopt-report=5 -qopt-report-phase=vec,loop")
   if ("${CMAKE_C_COMPILER_VERSION}" VERSION_GREATER "17.0.4")
     set(CMAKE_C_FLAGS_RELEASE "-qopt-zmm-usage=high ${CMAKE_C_FLAGS_RELEASE}")
   endif ("${CMAKE_C_COMPILER_VERSION}" VERSION_GREATER "17.0.4")
elseif ("${CMAKE_C_COMPILER_ID}" STREQUAL "MSVC")
   # using Visual Studio C++
   set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g -O0")
   set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -g -O3")
elseif (CMAKE_C_COMPILER_ID MATCHES "PGI")
elseif (CMAKE_C_COMPILER_ID MATCHES "Cray")
elseif (CMAKE_C_COMPILER_ID MATCHES "XL")
   set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
   set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -g -O0")
   set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -g -O3")
endif()

if (NOT DEFINED OpenCL_FOUND)
   find_package(OpenCL)
   #  OpenCL_FOUND          - True if OpenCL was found
   #  OpenCL_INCLUDE_DIRS   - include directories for OpenCL
   #  OpenCL_LIBRARIES      - link against this library to use OpenCL
   #  OpenCL_VERSION_STRING - Highest supported OpenCL version (eg. 1.2)
   #  OpenCL_VERSION_MAJOR  - The major version of the OpenCL implementation
   #  OpenCL_VERSION_MINOR  - The minor version of the OpenCL implementation
   if (OpenCL_FOUND)
      set(HAVE_CL_DOUBLE ON CACHE BOOL "Have OpenCL Double")
      set(NO_CL_DOUBLE OFF)
      include_directories(${OpenCL_INCLUDE_DIRS})
   endif (OpenCL_FOUND)
endif (NOT DEFINED OpenCL_FOUND)

if (NOT DEFINED MPI_FOUND)
   find_package(MPI)
endif (NOT DEFINED MPI_FOUND)
if (MPI_FOUND)
   include_directories(${MPI_INCLUDE_PATH})
endif (MPI_FOUND)

if (NOT DEFINED OPENMP_FOUND)
   find_package( OpenMP)
endif (NOT DEFINED OPENMP_FOUND)

########### global settings ###############
set(H_SRCS hash.h)

set(C_SRCS hash.c)

set(hash_LIB_SRCS ${C_SRCS} ${H_SRCS})

include_directories(${CMAKE_CURRENT_BINARY_DIR})

########### embed source target ###############
add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/hashlib_kern.inc
                  COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/embed_source.pl ${CMAKE_CURRENT_SOURCE_DIR}/hashlib_kern.cl > hashlib_kern.inc
                  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/hashlib_kern.cl ${CMAKE_CURRENT_SOURCE_DIR}/embed_source.pl)
add_custom_target(hashlib_kernel_source ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/hashlib_kern.inc)

add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/hashlib_source_kern.inc
                  COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/embed_source.pl ${CMAKE_CURRENT_SOURCE_DIR}/hashlib_source_kern.cl > hashlib_source_kern.inc
                  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/hashlib_source_kern.cl ${CMAKE_CURRENT_SOURCE_DIR}/embed_source.pl)
add_custom_target(hashlib_source_kernel_source ALL DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/hashlib_source_kern.inc)

########### hash target ###############

add_library(hash STATIC ${hash_LIB_SRCS})

set_target_properties(hash PROPERTIES VERSION ${HASH_VERSION} SOVERSION ${HASH_VERSION_MAJOR})
install(TARGETS hash DESTINATION lib)

########### mphash target ###############
if (OPENMP_FOUND)
   add_library(mphash STATIC ${hash_LIB_SRCS})

   set_target_properties(mphash PROPERTIES VERSION ${HASH_VERSION} SOVERSION ${HASH_VERSION_MAJOR})
   set_target_properties(mphash PROPERTIES COMPILE_FLAGS ${OpenMP_C_FLAGS})
   install(TARGETS mphash DESTINATION lib)
endif (OPENMP_FOUND)

########### dhash target ###############

if (OpenCL_FOUND)
   add_library(dhash STATIC ${hash_LIB_SRCS})

   set_target_properties(dhash PROPERTIES VERSION ${HASH_VERSION} SOVERSION ${HASH_VERSION_MAJOR})
   set_target_properties(dhash PROPERTIES COMPILE_DEFINITIONS "HAVE_OPENCL")
   install(TARGETS dhash DESTINATION lib)
   add_dependencies(dhash hashlib_source_kernel_source)
   add_dependencies(dhash hashlib_kernel_source)
endif(OpenCL_FOUND)

########### install files ###############

install(FILES  hash.h DESTINATION include/hash)

########### doc targets ################

if (${CMAKE_PROJECT_NAME} MATCHES ${PROJECT_NAME})
   add_custom_target(doc
                     COMMAND make pdfdoc
                     COMMAND make htmldoc
                     COMMAND make singlehtmldoc)
   set_target_properties(doc PROPERTIES EXCLUDE_FROM_ALL TRUE)

   add_custom_command(OUTPUT _build/latex/Hash.pdf
                      WORKING_DIRECTORY docs
                      COMMAND make latexpdf >& pdfdoc.out
                      DEPENDS ${PDFDOC_SRCS})
   add_custom_target(pdfdoc DEPENDS _build/latex/Hash.pdf)
   set_target_properties(pdfdoc PROPERTIES EXCLUDE_FROM_ALL TRUE)

   add_custom_command(OUTPUT _build/html/UsersGuide.html
                      WORKING_DIRECTORY docs
                      COMMAND make html >& htmldoc.out
                      DEPENDS ${DOC_SRCS})
   add_custom_target(htmldoc DEPENDS _build/html/UsersGuide.html)
   set_target_properties(htmldoc PROPERTIES EXCLUDE_FROM_ALL TRUE)

   add_custom_command(OUTPUT _build/singlehtml/index.html
                      WORKING_DIRECTORY docs
                      COMMAND make singlehtml >& singlehtmldoc.out
                      DEPENDS ${DOC_SRCS})
   add_custom_target(singlehtmldoc DEPENDS _build/singlehtml/index.html)
   set_target_properties(singlehtmldoc PROPERTIES EXCLUDE_FROM_ALL TRUE)
else (${CMAKE_PROJECT_NAME} MATCHES ${PROJECT_NAME})
   add_custom_target(Hash_doc
                     COMMAND make Hash_pdfdoc
                     COMMAND make Hash_htmldoc
                     COMMAND make Hash_singlehtmldoc)
   set_target_properties(Hash_doc PROPERTIES EXCLUDE_FROM_ALL TRUE)

   add_custom_command(OUTPUT _build/latex/Hash.pdf
                      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/docs
                      COMMAND make latexpdf >& pdfdoc.out
                      DEPENDS ${PDFDOC_SRCS})
   add_custom_target(Hash_pdfdoc DEPENDS _build/latex/Hash.pdf)
   set_target_properties(Hash_pdfdoc PROPERTIES EXCLUDE_FROM_ALL TRUE)

   add_custom_command(OUTPUT _build/html/UsersGuide.html
                      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/docs
                      COMMAND make html >& htmldoc.out
                      DEPENDS ${DOC_SRCS})
   add_custom_target(Hash_htmldoc DEPENDS _build/html/UsersGuide.html)
   set_target_properties(Hash_htmldoc PROPERTIES EXCLUDE_FROM_ALL TRUE)

   add_custom_command(OUTPUT _build/singlehtml/index.html
                      WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/docs
                      COMMAND make singlehtml >& singlehtmldoc.out
                      DEPENDS ${DOC_SRCS})
   add_custom_target(Hash_singlehtmldoc DEPENDS _build/singlehtml/index.html)
   set_target_properties(Hash_singlehtmldoc PROPERTIES EXCLUDE_FROM_ALL TRUE)
endif (${CMAKE_PROJECT_NAME} MATCHES ${PROJECT_NAME})

########### clean files ################
SET_DIRECTORY_PROPERTIES(PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES
   "install_manifest.txt;docs/_build;docs/htmldoc.out;docs/pdfdoc.out;docs/singlehtmldoc.out")

