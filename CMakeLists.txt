cmake_minimum_required(VERSION 3.12)
project("bert.cpp" C CXX)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

#
# Option list
#

# general
option(BERT_STATIC                 "bert: static link libraries"                          OFF)
option(BERT_NATIVE                 "bert: enable -march=native flag"                       OFF)
option(BERT_LTO                    "bert: enable link time optimization"                  OFF)

# debug
option(BERT_ALL_WARNINGS           "bert: enable all compiler warnings"                   OFF)
option(BERT_ALL_WARNINGS_3RD_PARTY "bert: enable all compiler warnings in 3rd party libs" OFF)
option(BERT_GPROF                  "bert: enable gprof"                                   OFF)
option(BERT_PERF                   "bert: enable performance logging"                     OFF)

# sanitizers
option(BERT_SANITIZE_THREAD        "bert: enable thread sanitizer"                        OFF)
option(BERT_SANITIZE_ADDRESS       "bert: enable address sanitizer"                       OFF)
option(BERT_SANITIZE_UNDEFINED     "bert: enable undefined sanitizer"                     OFF)

# instruction set specific
option(BERT_AVX                    "bert: enable AVX"                                     ON)
option(BERT_AVX2                   "bert: enable AVX2"                                    ON)
option(BERT_FMA                    "bert: enable FMA"                                     ON)

# 3rd party libs
option(BERT_ACCELERATE             "bert: enable Accelerate framework"                    ON)
option(BERT_OPENBLAS               "bert: use OpenBLAS"                                   OFF)

#
# Compile flags
#

set(CMAKE_CXX_STANDARD_REQUIRED true)
set(CMAKE_C_STANDARD_REQUIRED true)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

if (NOT MSVC)
    if (BERT_SANITIZE_THREAD)
        add_compile_options(-fsanitize=thread)
        link_libraries(-fsanitize=thread)
    endif()

    if (BERT_SANITIZE_ADDRESS)
        add_compile_options(-fsanitize=address -fno-omit-frame-pointer)
        link_libraries(-fsanitize=address)
    endif()

    if (BERT_SANITIZE_UNDEFINED)
        add_compile_options(-fsanitize=undefined)
        link_libraries(-fsanitize=undefined)
    endif()
endif()

if (APPLE AND BERT_ACCELERATE)
    find_library(ACCELERATE_FRAMEWORK Accelerate)
    if (ACCELERATE_FRAMEWORK)
        message(STATUS "Accelerate framework found")

        add_compile_definitions(GGML_USE_ACCELERATE)
        set(BERT_EXTRA_LIBS ${BERT_EXTRA_LIBS} ${ACCELERATE_FRAMEWORK})
    else()
        message(WARNING "Accelerate framework not found")
    endif()
endif()
if (BERT_OPENBLAS)
    if (BERT_STATIC)
        set(BLA_STATIC ON)
    endif()

    set(BLA_VENDOR OpenBLAS)
    find_package(BLAS)
    if (BLAS_FOUND)
        message(STATUS "OpenBLAS found")

        add_compile_definitions(GGML_USE_OPENBLAS)
        add_link_options(${BLAS_LIBRARIES})
    else()
        message(WARNING "OpenBLAS not found")
    endif()
endif()

if (BERT_ALL_WARNINGS)
    if (NOT MSVC)
        add_compile_options(
            -Wall
            -Wextra
            -Wpedantic
            -Wcast-qual
            -Wno-unused-function
        )
    else()
        # todo : msvc
    endif()
endif()

if(NOT WIN32)
add_compile_options(-Wno-format)
endif()

if(BERT_PERF)
    add_compile_definitions(GGML_PERF)
endif()

if (NOT MSVC)
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O3")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3")
else()
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} /Ox")
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /Ox")
endif()

if (BERT_LTO)
    include(CheckIPOSupported)
    check_ipo_supported(RESULT result OUTPUT output)
    if (result)
        set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
    else()
        message(WARNING "IPO is not supported: ${output}")
    endif()
endif()

# Architecture specific
# TODO: probably these flags need to be tweaked on some architectures
#       feel free to update the Makefile for your architecture and send a pull request or issue
message(STATUS "CMAKE_SYSTEM_PROCESSOR: ${CMAKE_SYSTEM_PROCESSOR}")
if (NOT MSVC)
    if (BERT_STATIC)
        add_link_options(-static)
        if (MINGW)
            add_link_options(-static-libgcc -static-libstdc++)
        endif()
    endif()
    if (BERT_GPROF)
        add_compile_options(-pg)
    endif()
    if (BERT_NATIVE)
        add_compile_options(-march=native)
    endif()
endif()

if (${CMAKE_SYSTEM_PROCESSOR} MATCHES "arm" OR ${CMAKE_SYSTEM_PROCESSOR} MATCHES "aarch64")
    message(STATUS "ARM detected")
    if (MSVC)
        # TODO: arm msvc?
    else()
        if (${CMAKE_SYSTEM_PROCESSOR} MATCHES "aarch64")
            add_compile_options(-mcpu=native)
        endif()
        # TODO: armv6,7,8 version specific flags
    endif()
elseif (${CMAKE_SYSTEM_PROCESSOR} MATCHES "^(x86_64|i686|AMD64)$")
    message(STATUS "x86 detected")
    if (NOT MSVC)
        add_compile_options(-mf16c)
        if (BERT_FMA)
            add_compile_options(-mfma)
        endif()
        if (BERT_AVX)
            add_compile_options(-mavx)
        endif()
        if (BERT_AVX2)
            add_compile_options(-mavx2)
        endif()
    endif()
else()
    # TODO: support PowerPC
    message(STATUS "Unknown architecture")
endif()

#
# Build libraries
#

if (MSVC)
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
endif()

set(TOKENZIER_CPP_PATH tokenizers-cpp)
add_subdirectory(${TOKENZIER_CPP_PATH} tokenizers)

include(CheckCXXCompilerFlag)
if(NOT MSVC)
  check_cxx_compiler_flag("-std=c++17" SUPPORT_CXX17)
  set(CMAKE_CXX_FLAGS "-std=c++17 ${CMAKE_CXX_FLAGS}")
  set(CMAKE_CUDA_STANDARD 17)
else()
  check_cxx_compiler_flag("/std:c++17" SUPPORT_CXX17)
  set(CMAKE_CXX_FLAGS "/std:c++17 ${CMAKE_CXX_FLAGS}")
  set(CMAKE_CUDA_STANDARD 17)
endif()


add_subdirectory(ggml)

# List of library names
set(LIBRARIES bert bert_shared)
set(BERT_SOURCE 
    bert.cpp
    bert.h
    tokenizer.cpp
    tokenizer.h    
    gguf.h
)

add_library(bert
            ${BERT_SOURCE}
            )
add_library(bert_shared
            SHARED
            ${BERT_SOURCE}
            )

# Loop over the library names
foreach(LIB ${LIBRARIES})
    
target_include_directories(${LIB} PRIVATE ${TOKENZIER_CPP_PATH}/include)

target_include_directories(${LIB} PUBLIC .)
target_compile_features(${LIB} PUBLIC cxx_std_20)
target_link_libraries(${LIB} PRIVATE ggml ${BERT_EXTRA_LIBS})

target_link_libraries(${LIB} PRIVATE tokenizers_cpp)
endforeach()

if(WIN32)
include_directories(${CMAKE_SOURCE_DIR}/include_win)
endif()

set_target_properties(ggml PROPERTIES POSITION_INDEPENDENT_CODE ON)
set_target_properties(bert_shared PROPERTIES POSITION_INDEPENDENT_CODE ON)
target_compile_definitions(bert_shared PRIVATE BERT_SHARED BERT_BUILD)

add_subdirectory(examples)
add_subdirectory(models)