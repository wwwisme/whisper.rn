cmake_minimum_required(VERSION 3.10)

project(whisper.rn)

set(CMAKE_CXX_STANDARD 17)
set(RNWHISPER_LIB_DIR ${CMAKE_SOURCE_DIR}/../../../cpp)

set(
    SOURCE_FILES
    ${RNWHISPER_LIB_DIR}/ggml.c
    ${RNWHISPER_LIB_DIR}/ggml-alloc.c
    ${RNWHISPER_LIB_DIR}/ggml-backend.cpp
    ${RNWHISPER_LIB_DIR}/ggml-backend-reg.cpp
    ${RNWHISPER_LIB_DIR}/ggml-cpu.c
    ${RNWHISPER_LIB_DIR}/ggml-cpu.cpp
    ${RNWHISPER_LIB_DIR}/ggml-cpu-aarch64.cpp
    ${RNWHISPER_LIB_DIR}/ggml-cpu-quants.c
    ${RNWHISPER_LIB_DIR}/ggml-cpu-traits.cpp
    ${RNWHISPER_LIB_DIR}/ggml-opt.cpp
    ${RNWHISPER_LIB_DIR}/ggml-threading.cpp
    ${RNWHISPER_LIB_DIR}/ggml-quants.c
    ${RNWHISPER_LIB_DIR}/gguf.cpp
    ${RNWHISPER_LIB_DIR}/amx/amx.cpp
    ${RNWHISPER_LIB_DIR}/amx/mmq.cpp
    ${RNWHISPER_LIB_DIR}/whisper.cpp
    ${RNWHISPER_LIB_DIR}/rn-audioutils.cpp
    ${RNWHISPER_LIB_DIR}/rn-whisper.cpp
    ${CMAKE_SOURCE_DIR}/jni.cpp
)

find_library(LOG_LIB log)

function(build_library target_name)
    add_library(
        ${target_name}
        SHARED
        ${SOURCE_FILES}
    )

    target_link_libraries(${target_name} ${LOG_LIB} android)

    target_compile_options(${target_name} PRIVATE -DWSP_GGML_USE_CPU -DWSP_GGML_USE_CPU_AARCH64)

    if (${target_name} STREQUAL "whisper_v8fp16_va")
        target_compile_options(${target_name} PRIVATE -march=armv8.2-a+fp16)
    elseif (${target_name} STREQUAL "whisper_vfpv4")
        target_compile_options(${target_name} PRIVATE -mfpu=neon-vfpv4)
    endif ()

    if (${CMAKE_BUILD_TYPE} STREQUAL "Debug")
        target_compile_options(${target_name} PRIVATE -DRNWHISPER_ANDROID_ENABLE_LOGGING)
    endif ()

    # NOTE: If you want to debug the native code, you can uncomment if and endif
    # if (NOT ${CMAKE_BUILD_TYPE} STREQUAL "Debug")

        target_compile_options(${target_name} PRIVATE -O3 -DNDEBUG -pthread)
        target_compile_options(${target_name} PRIVATE -fvisibility=hidden -fvisibility-inlines-hidden)
        target_compile_options(${target_name} PRIVATE -ffunction-sections -fdata-sections)

        target_link_options(${target_name} PRIVATE -Wl,--gc-sections)
        target_link_options(${target_name} PRIVATE -Wl,--exclude-libs,ALL)
        target_link_options(${target_name} PRIVATE -flto)

    # endif ()
endfunction()

build_library("whisper") # Default target

if (${ANDROID_ABI} STREQUAL "arm64-v8a")
    build_library("whisper_v8fp16_va")
elseif (${ANDROID_ABI} STREQUAL "armeabi-v7a")
    build_library("whisper_vfpv4")
endif ()

include_directories(${RNWHISPER_LIB_DIR})
