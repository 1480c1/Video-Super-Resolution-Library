cmake_minimum_required(VERSION 3.14)
project(Raisr CXX)

if("${CMAKE_CURRENT_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_BINARY_DIR}")
    message(WARNING "Building in-source is highly not recommended\n"
                    "Please use the Build folder or create your own.")
endif()


set(CMAKE_BUILD_TYPE Release)
set(CMAKE_CXX_STANDARD 17)

option(BUILD_SHARED_LIBS "Build shared libraries" ON)

include(CheckCXXCompilerFlag)
include(CheckCXXSourceCompiles)
include(GNUInstallDirs)

if(DEFINED ENV{IPPROOT})
    message(STATUS "Using IPPROOT from environment")
    set(CMAKE_PREFIX_PATH $ENV{IPPROOT}/lib/cmake/ipp ${CMAKE_PREFIX_PATH})
endif()

set(RAISR_VERSION_MAJOR           "22")
set(RAISR_VERSION_MINOR           "12")
configure_file(Library/RaisrVersion.h.in RaisrVersion.h @ONLY)

if(MSVC)
    set(AVX512_FLAGS "/arch:AVX512")
else()
    set(AVX512_FLAGS "-mavx512f -mavx512vl -mavx512dq") # generic flag for gcc, clang, icc
endif()

set(CMAKE_REQUIRED_FLAGS_OLD ${CMAKE_REQUIRED_FLAGS})
set(CMAKE_REQUIRED_FLAGS ${AVX512_FLAGS})
check_cxx_source_compiles("
    #include <immintrin.h>
    int main() {
        float data[16] = {0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15};
        float output[16];
        __m512 a = _mm512_loadu_ps(data); // avx512f
        __m512 b = _mm512_loadu_ps(data);
        __mmask8 mask = _mm512_cmp_ps_mask(a, b, _CMP_EQ_OQ); // avx512vl
        (void)mask;
        __m512 c = _mm512_add_ps(a, b); // avx512f
        _mm512_storeu_ps(output, c); // avx512f
        return 0;
    }" HAVE_AVX512)
set(CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS_OLD})

# Intel Library for Video Super Resolution
set(public_headers
    Library/Raisr_AVX256.h
    Library/RaisrDefaults.h
    Library/Raisr_globals.h
    Library/Raisr.h
    Library/RaisrHandler.h
    Library/ThreadPool.h)

add_library(raisr
    Library/Raisr_AVX256.cpp
    Library/Raisr.cpp
    Library/RaisrHandler.cpp
    ${public_headers})

set_target_properties(raisr PROPERTIES PUBLIC_HEADER "${public_headers}")

check_cxx_compiler_flag(-mfma HAVE_FMA)
if(HAVE_FMA)
    target_compile_options(raisr PRIVATE -mfma)
endif()
if(MSVC)
    set(AVX2_FLAG "/arch:AVX2")
else()
    set(AVX2_FLAG "-mavx2")
endif()
check_cxx_compiler_flag(${AVX2_FLAG} HAVE_AVX2)
if(HAVE_AVX2)
    target_compile_options(raisr PRIVATE ${AVX2_FLAG})
endif()

if(HAVE_AVX512)
    message(STATUS "Building with AVX512")
    target_sources(raisr PRIVATE
        Library/Raisr_AVX512.cpp
        Library/Raisr_AVX512.h)
    string(REPLACE " " ";" AVX512_FLAGS_LIST ${AVX512_FLAGS})
    target_compile_options(raisr PRIVATE ${AVX512_FLAGS_LIST})
else()
    message(STATUS "Building without AVX512")
endif()

# Raisr can use SVML for atan2 if Intel Compiler can be found, otherwise uses an atanApproximation
if(CMAKE_CXX_COMPILER_ID STREQUAL "Intel")
    message(STATUS "Building with Intel Compiler, using SVML")
else()
    message(STATUS "Building with Atan Approximation")
    target_compile_definitions(raisr PRIVATE USE_ATAN2_APPROX)
endif()

find_package(IPP REQUIRED)
find_package(Threads REQUIRED)
target_link_libraries(raisr ${IPP_LIBRARIES} Threads::Threads)
target_include_directories(raisr PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

install(TARGETS raisr
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    PUBLIC_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/raisr")
