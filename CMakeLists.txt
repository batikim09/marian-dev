cmake_minimum_required(VERSION 3.5.1)
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

project(marian CXX C)
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Custom CMake options
option(COMPILE_EXAMPLES "Compile examples" OFF)
option(COMPILE_TESTS "Compile tests" OFF)
option(COMPILE_SERVER "Compile marian-server" ON)
option(COMPILE_CPU "Compile CPU version" ON)
option(COMPILE_CUDA "Compile GPU version" ON)
option(USE_STATIC_LIBS "Compile GPU version" OFF)
option(USE_CUDNN "Use CUDNN library" OFF)
option(USE_NCCL "Use NCCL library" ON)
option(USE_MPI "Use MPI library" OFF)
option(USE_SENTENCEPIECE "Download and compile SentencePiece" OFF)

# Project versioning
find_package(Git QUIET)
include(GetVersionFromFile)

message(STATUS "Project name: ${PROJECT_NAME}")
message(STATUS "Project version: ${PROJECT_VERSION_STRING_FULL}")

execute_process(COMMAND git submodule update --init --recursive --no-fetch
                WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR})

# Set compilation flags
if(MSVC)
  set(CMAKE_CXX_FLAGS           "/EHsc /DWIN32 /D_WINDOWS /DUNICODE /D_UNICODE /D_CRT_NONSTDC_NO_WARNINGS /D_CRT_SECURE_NO_WARNINGS")
  set(CMAKE_CXX_FLAGS_RELEASE   "/MT /O2 /W4 /Zi /MP /GL /DNDEBUG")
  set(CMAKE_CXX_FLAGS_DEBUG     "/MTd /Od /Ob0 /RTC1 /Zi /D_DEBUG")

  set(CMAKE_EXE_LINKER_FLAGS    "${CMAKE_EXE_LINKER_FLAGS} /DEBUG /LTCG:incremental /INCREMENTAL:NO")

  set(CMAKE_STATIC_LINKER_FLAGS "${CMAKE_STATIC_LINKER_FLAGS} /LTCG:incremental")
else()
  set(CMAKE_CXX_FLAGS_RELEASE         " -std=c++11 -O3 -Ofast -m64 -pthread -march=native -Wl,--no-as-needed -funroll-loops -ffinite-math-only -fPIC -Wno-unused-result -Wno-deprecated -Wno-deprecated-gpu-targets")
  set(CMAKE_CXX_FLAGS_NONATIVE        " -std=c++11 -O3 -Ofast -m64 -pthread -march=x86-64 -mavx -Wl,--no-as-needed -funroll-loops -ffinite-math-only -fPIC -Wno-unused-result -Wno-deprecated -Wno-deprecated-gpu-targets")
  set(CMAKE_CXX_FLAGS_DEBUG           " -std=c++11 -g -rdynamic -O0 -pthread -fPIC -Wno-unused-result -Wno-deprecated -Wno-deprecated-gpu-targets")
  set(CMAKE_CXX_FLAGS_ST              "${CMAKE_CXX_FLAGS_RELEASE} -DNDEBUG")
  set(CMAKE_CXX_FLAGS_RELWITHDEBINFO  "${CMAKE_CXX_FLAGS_RELEASE} -pg -g -rdynamic")
  set(CMAKE_CXX_FLAGS_PROFILE         "${CMAKE_CXX_FLAGS_RELEASE} -pg -g -rdynamic")
  set(CMAKE_CXX_FLAGS_PROFGEN         "${CMAKE_CXX_FLAGS_RELEASE} -fprofile-generate -fprofile-correction")
  set(CMAKE_CXX_FLAGS_PROFUSE         "${CMAKE_CXX_FLAGS_RELEASE} -fprofile-use -fprofile-correction")
  set(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS_RELEASE})
endif()

# Downloading SentencePiece if requested and set to compile with it.
# Requires all the dependencies imposed by SentencePiece
if(USE_SENTENCEPIECE)
  message(STATUS "Using SentencePiece from our fork https://github.com/marian-nmt/sentencepiece.git")
  if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/src/3rd_party/sentencepiece)
    execute_process(COMMAND git clone https://github.com/marian-nmt/sentencepiece.git
                    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/src/3rd_party
                    RESULT_VARIABLE git_result
                    ERROR_QUIET)
    message(STATUS "Downloaded SentencePiece [code: ${git_result}]")
  else()
    message(STATUS "It seems that SentencePiece has already been downloaded. Reusing.")
  endif()

  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DUSE_SENTENCEPIECE")
  LIST(APPEND CUDA_NVCC_FLAGS -DUSE_SENTENCEPIECE; )
  set(EXT_LIBS ${EXT_LIBS} sentencepiece)
endif()


# Find packages
set(EXT_LIBS ${EXT_LIBS} ${CMAKE_DL_LIBS})

if(COMPILE_CUDA)
find_package(CUDA "8.0")
if(CUDA_FOUND)
  set(EXT_LIBS ${EXT_LIBS} ${CUDA_curand_LIBRARY} ${CUDA_cusparse_LIBRARY})

  if(USE_CUDNN)
    find_package(CUDNN "7.0")
    if(CUDNN_FOUND)
      include_directories(${CUDNN_INCLUDE_DIRS})
      set(EXT_LIBS ${EXT_LIBS} ${CUDNN_LIBRARIES})
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCUDNN")
      LIST(APPEND CUDA_NVCC_FLAGS -DCUDNN; )
    endif(CUDNN_FOUND)
  endif(USE_CUDNN)

  if(USE_NCCL)
    find_package(NCCL)
    if(NCCL_FOUND)
      include_directories(${NCCL_INCLUDE_DIR})
      set(EXT_LIBS ${EXT_LIBS} ${NCCL_LIBRARIES})
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DUSE_NCCL")
      LIST(APPEND CUDA_NVCC_FLAGS -DUSE_NCCL; )
    endif(NCCL_FOUND)
  endif(USE_NCCL)

  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DCUDA_FOUND")
  list(APPEND CUDA_NVCC_FLAGS -DCUDA_FOUND; )

  if(MSVC)
    list(APPEND CUDA_NVCC_FLAGS -DBOOST_PP_VARIADICS=0; )
  endif()

else(CUDA_FOUND)
  message(FATAL_ERROR "CUDA has not been found, set -DCOMPILE_CUDA=off to avoid this check and to compile the CPU version only")
endif(CUDA_FOUND)

else(COMPILE_CUDA)
  message(WARNING "COMPILE_CUDA=off : Building only CPU version")
endif(COMPILE_CUDA)

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
  list(APPEND CUDA_NVCC_FLAGS --default-stream per-thread; -O0; -g; -arch=sm_30; -gencode=arch=compute_30,code=sm_30; -gencode=arch=compute_50,code=sm_50; -gencode=arch=compute_52,code=sm_52; -gencode=arch=compute_60,code=sm_60; -gencode=arch=compute_61,code=sm_61; -gencode=arch=compute_61,code=compute_61 ;)
else(CMAKE_BUILD_TYPE STREQUAL "Debug")
  list(APPEND CUDA_NVCC_FLAGS --default-stream per-thread; -O3; -g; --use_fast_math; -arch=sm_30; -gencode=arch=compute_30,code=sm_30; -gencode=arch=compute_50,code=sm_50; -gencode=arch=compute_52,code=sm_52; -gencode=arch=compute_60,code=sm_60; -gencode=arch=compute_61,code=sm_61; -gencode=arch=compute_61,code=compute_61 ;)
endif(CMAKE_BUILD_TYPE STREQUAL "Debug")
if(NOT MSVC)
  list(APPEND CUDA_NVCC_FLAGS -std=c++11; -Xcompiler\ -fPIC; )
else()
  list(APPEND CUDA_NVCC_FLAGS -Xcompiler\ /FS; )
endif()

list(REMOVE_DUPLICATES CUDA_NVCC_FLAGS)
set(CUDA_PROPAGATE_HOST_FLAGS OFF)

if(USE_STATIC_LIBS)
  set(_ORIG_CMAKE_FIND_LIBRARY_SUFFIXES ${CMAKE_FIND_LIBRARY_SUFFIXES})
  if(WIN32)
    list(INSERT CMAKE_FIND_LIBRARY_SUFFIXES 0 .lib .a)
  else()
    set(CMAKE_FIND_LIBRARY_SUFFIXES .a)
  endif()
endif()

if(NOT WIN32)
  find_package(Tcmalloc)
  if(Tcmalloc_FOUND)
    include_directories(${Tcmalloc_INCLUDE_DIR})
    set(EXT_LIBS ${EXT_LIBS} ${Tcmalloc_LIBRARIES})
  else(Tcmalloc_FOUND)
    message(WARNING "Cannot find TCMalloc library. Continuing.")
  endif(Tcmalloc_FOUND)
endif()

if(USE_MPI)
  find_package(MPI 2.0)
  if(MPI_FOUND)
    include_directories(${MPI_INCLUDE_PATH})
    set(EXT_LIBS ${EXT_LIBS} ${MPI_LIBRARIES})
    add_definitions(-DMPI_FOUND=1)
  endif(MPI_FOUND)
endif(USE_MPI)

if(COMPILE_CPU)
  find_package(MKL)
  if(MKL_FOUND)
    include_directories(${MKL_INCLUDE_DIR})
    set(EXT_LIBS ${EXT_LIBS} ${MKL_LIBRARIES})
    add_definitions(-DBLAS_FOUND=1 -DMKL_FOUND=1)
  else(MKL_FOUND)
    set(BLA_VENDOR "OpenBLAS")
    find_package(BLAS)
    if(BLAS_FOUND)
      include_directories(${BLAS_INCLUDE_DIR})
      set(EXT_LIBS ${EXT_LIBS} ${BLAS_LIBRARIES})
      add_definitions(-DBLAS_FOUND=1)
    endif(BLAS_FOUND)
  endif(MKL_FOUND)
endif(COMPILE_CPU)

set(BOOST_COMPONENTS timer iostreams filesystem system)
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.9)
  add_definitions(-DUSE_BOOST_REGEX=1)
  set(BOOST_COMPONENTS ${BOOST_COMPONENTS} regex)
  message("-- Using boost::regex")
else()
  message("-- Using std::regex")
endif()

if(COMPILE_SERVER)
  find_package(OpenSSL)
  if(OpenSSL_FOUND)
    message(STATUS "Found OpenSSL")
    include_directories(${OPENSSL_INCLUDE_DIR})
    set(EXT_LIBS ${EXT_LIBS} ${OPENSSL_CRYPTO_LIBRARY})
  else(OpenSSL_FOUND)
    message(WARNING "Cannot find OpenSSL library. Not compiling server.")
    set(COMPILE_SERVER "off")
  endif(OpenSSL_FOUND)
endif(COMPILE_SERVER)

find_package(ZLIB)
if(ZLIB_FOUND)
  include_directories(${ZLIB_INCLUDE_DIRS})
  set(EXT_LIBS ${EXT_LIBS} ${ZLIB_LIBRARIES})
else(ZLIB_FOUND)
  message(SEND_ERROR "Cannot find zlib.")
endif(ZLIB_FOUND)

if(USE_STATIC_LIBS)
  set(CMAKE_FIND_LIBRARY_SUFFIXES ${_ORIG_CMAKE_FIND_LIBRARY_SUFFIXES})
endif()

if(USE_STATIC_LIBS)
  set(Boost_USE_STATIC_LIBS ON)
endif()

find_package(Boost COMPONENTS ${BOOST_COMPONENTS})
if(Boost_FOUND)
  include_directories(${Boost_INCLUDE_DIRS})
  set(EXT_LIBS ${EXT_LIBS} ${Boost_LIBRARIES})
else(Boost_FOUND)
  message(SEND_ERROR "Cannot find Boost libraries. Terminating.")
endif(Boost_FOUND)

if(COMPILE_TESTS)
  enable_testing()
endif(COMPILE_TESTS)

if(COMPILE_EXAMPLES)
  add_definitions(-DCOMPILE_EXAMPLES=1)
endif(COMPILE_EXAMPLES)

# Compile source files
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/common/version.h.in
               ${CMAKE_CURRENT_SOURCE_DIR}/src/common/version.h @ONLY)

include_directories(${marian_SOURCE_DIR}/src)
add_subdirectory(src)


# Add a target to generate API documentation with Doxygen
find_package(Doxygen)
if(DOXYGEN_FOUND)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in
           ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile @ONLY)
  add_custom_target(doc
    ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Generating API documentation with Doxygen" VERBATIM
  )
endif(DOXYGEN_FOUND)

