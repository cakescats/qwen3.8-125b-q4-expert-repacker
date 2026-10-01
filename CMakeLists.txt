cmake_minimum_required(VERSION 3.20)
project(qwen-expert-repacker CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

if(WIN32)
  message(FATAL_ERROR "qwfn-pack uses POSIX file I/O (O_DIRECT, pread/pwrite, posix_fallocate); build it on Linux.")
endif()

# The GGUF reader is ggml's (libggml-base). Any llama.cpp build provides it; the
# default is where QwFNfer's README puts one.
set(LLAMA_CPP_ROOT "$ENV{HOME}/.unsloth/llama.cpp" CACHE PATH "llama.cpp source tree")
set(LLAMA_CPP_BUILD "${LLAMA_CPP_ROOT}/build/bin" CACHE PATH "directory holding libggml-base.so")
if(NOT EXISTS "${LLAMA_CPP_ROOT}/ggml/include/gguf.h")
  message(FATAL_ERROR "ggml headers not found under ${LLAMA_CPP_ROOT}. Pass -DLLAMA_CPP_ROOT=<llama.cpp>.")
endif()
find_library(GGML_BASE NAMES ggml-base PATHS "${LLAMA_CPP_BUILD}" "${LLAMA_CPP_ROOT}/build/lib" NO_DEFAULT_PATH)
if(NOT GGML_BASE)
  message(FATAL_ERROR "libggml-base not found under ${LLAMA_CPP_BUILD}. Build llama.cpp, or pass -DLLAMA_CPP_BUILD=<dir>.")
endif()
find_package(Threads REQUIRED)

add_executable(qwfn-pack
  tools/qwfn_pack.cpp
  src/qwfn_pack.cpp
  src/qwfn_model.cpp
  src/qwfn_plat_posix.cpp)
target_include_directories(qwfn-pack PRIVATE src "${LLAMA_CPP_ROOT}/ggml/include")
target_link_libraries(qwfn-pack PRIVATE "${GGML_BASE}" Threads::Threads)
target_compile_options(qwfn-pack PRIVATE -O2 -Wall -Wextra -Wno-unused-parameter)
set_target_properties(qwfn-pack PROPERTIES BUILD_RPATH "${LLAMA_CPP_BUILD}")
