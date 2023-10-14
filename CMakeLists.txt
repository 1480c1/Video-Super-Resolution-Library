cmake_minimum_required(VERSION 3.14)
project(Raisr CXX)

if("${CMAKE_CURRENT_SOURCE_DIR}" STREQUAL "${CMAKE_CURRENT_BINARY_DIR}")
    message(WARNING "Building in-source is highly not recommended\n"
                    "Please use the Build folder or create your own.")
endif()


set(CMAKE_BUILD_TYPE Release)
set(CMAKE_CXX_STANDARD 17)

include(CheckCXXCompilerFlag)

set(RAISR_VERSION_MAJOR           "22")
set(RAISR_VERSION_MINOR           "12")
configure_file(Library/RaisrVersion.h.in Library/RaisrVersion.h @ONLY)

if( UNIX )

set(flags_to_test
    -Wno-narrowing
    -fPIC
)

# Raisr can use SVML for atan2 if Intel Compiler can be found, otherwise uses an atanApproximation
if(CMAKE_CXX_COMPILER_ID STREQUAL "Intel")
    message(STATUS "Building with Intel Compiler, using SVML")
else()
    message(STATUS "Building with Atan Approximation")
    list(APPEND flags_to_test "-DUSE_ATAN2_APPROX")
endif()

INCLUDE(CheckCXXSourceRuns)
SET(CMAKE_REQUIRED_FLAGS "-march=native")
check_cxx_source_runs("
    #include <immintrin.h>
    int main(int argc, char** argv) {
        float data[16] = {0,1,2,3,4,5,6,7,8,9,10,11,12,13,14,15};
        float output[16];
        __m512 a = _mm512_loadu_ps(data); // avx512f
        __m512 b = _mm512_loadu_ps(data);
        __mmask8 mask = _mm512_cmp_ps_mask(a, b, _CMP_EQ_OQ); // avx512vl
        __m512 c = _mm512_add_ps(a, b); // avx512f
        _mm512_storeu_ps(output, c); // avx512f
        return 0;
    }" HAVE_AVX512)

foreach(cflag ${flags_to_test})
    string(REGEX REPLACE "[^A-Za-z0-9]" "_" cflag_var "${cflag}")
    set(test_cxx_flag "CXX_FLAG${cflag_var}")
    check_cxx_compiler_flag(${cflag} "${test_cxx_flag}")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${cflag}")
endforeach()

endif(UNIX)


# Intel Library for Video Super Resolution
add_subdirectory(Library)

# Raisr Sample app
include_directories(Library/)

