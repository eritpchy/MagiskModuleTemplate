cmake_minimum_required(VERSION 3.4.1)

if (NOT DEFINED MODULE_NAME)
    message(FATAL_ERROR "MODULE_NAME is not set")
else ()
    project(${MODULE_NAME})
endif ()

add_definitions(-DRIRU_MODULE)
add_definitions(-DMODULE_NAME="${MODULE_NAME}")

configure_file(template/config.cpp config.cpp)

message("Build type: ${CMAKE_BUILD_TYPE}")

set(CMAKE_CXX_STANDARD 11)

set(LINKER_FLAGS "-ffixed-x18 -Wl,--hash-style=both")
set(C_FLAGS "-Werror=format -fdata-sections -ffunction-sections")
set(CXX_FLAGS "${CXX_FLAGS} -fno-exceptions -fno-rtti")

if (NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
    set(C_FLAGS "${C_FLAGS} -O2 -fvisibility=hidden -fvisibility-inlines-hidden")
    set(LINKER_FLAGS "${LINKER_FLAGS} -Wl,-exclude-libs,ALL -Wl,--gc-sections -Wl,--strip-all")
else ()
    set(C_FLAGS "${C_FLAGS} -O0")
endif ()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${C_FLAGS}")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${C_FLAGS} ${CXX_FLAGS}")

set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${LINKER_FLAGS}")
set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} ${LINKER_FLAGS}")

find_package(riru REQUIRED CONFIG)

include_directories(include)

if(NOT DEFINED RESOURCE_BIN)
    message(FATAL_ERROR "RESOURCE_BIN未定义，请指定资源二进制文件路径")
endif()
set(RESOURCE_OBJ "${CMAKE_BINARY_DIR}/resource_${ANDROID_ABI}.o")

set(OBJCOPY_EXECUTABLE "${ANDROID_TOOLCHAIN_ROOT}/bin/llvm-objcopy${CMAKE_EXECUTABLE_SUFFIX}")

if(NOT EXISTS "${OBJCOPY_EXECUTABLE}")
    message(FATAL_ERROR "找不到NDK中的objcopy工具: ${OBJCOPY_EXECUTABLE}")
endif()

message(STATUS "Found objcopy: ${OBJCOPY_EXECUTABLE}")

function(binary_to_object input_file output_file)
    get_filename_component(input_filename ${input_file} NAME)
    set(local_copy "${input_filename}")
    if(ANDROID_ABI STREQUAL "arm64-v8a")
        set(ARCH_ARGS "--input-target=binary" "--output-target=elf64-littleaarch64" "--binary-architecture=aarch64")
    elseif(ANDROID_ABI STREQUAL "armeabi-v7a")
        set(ARCH_ARGS "--input-target=binary" "--output-target=elf32-littlearm" "--binary-architecture=arm")
    elseif(ANDROID_ABI STREQUAL "x86")
        set(ARCH_ARGS "--input-target=binary" "--output-target=elf32-i386" "--binary-architecture=i386")
    elseif(ANDROID_ABI STREQUAL "x86_64")
        set(ARCH_ARGS "--input-target=binary" "--output-target=elf64-x86-64" "--binary-architecture=i386:x86-64")
    endif()
    
    add_custom_command(
        OUTPUT ${output_file}
        COMMAND ${CMAKE_COMMAND} -E copy ${input_file} ${local_copy}
        COMMAND ${OBJCOPY_EXECUTABLE} ${ARCH_ARGS} 
                --rename-section .data=.,CONTENTS,ALLOC,LOAD,READONLY,DATA
                ${local_copy} ${output_file}
        DEPENDS ${input_file}
        COMMENT "Converting binary ${input_file} to object ${output_file}"
        VERBATIM
    )
endfunction()

binary_to_object(${RESOURCE_BIN} ${RESOURCE_OBJ})

add_custom_target(generate_resource_object ALL DEPENDS ${RESOURCE_OBJ})

find_program(CRC32_COMMAND crc32)

if(NOT CRC32_COMMAND)
    message(FATAL_ERROR "未找到crc32命令，请安装后再继续")
endif()

# 计算CRC32并将结果作为宏定义传递给编译器
execute_process(
    COMMAND ${CRC32_COMMAND} ${RESOURCE_BIN}
    RESULT_VARIABLE CRC32_RESULT
    OUTPUT_VARIABLE RESOURCE_CRC32
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

if(NOT CRC32_RESULT EQUAL 0)
    message(FATAL_ERROR "crc32命令执行失败: ${CRC32_ERROR}")
endif()

message(STATUS "计算得到的CRC32值: 0x${RESOURCE_CRC32}")

# 将CRC32值作为宏定义传递给编译器
add_definitions(-DRESOURCE_CRC32=0x${RESOURCE_CRC32})

add_library(${MODULE_NAME} SHARED 
    main.cpp 
    ${CMAKE_CURRENT_BINARY_DIR}/config.cpp
    ${RESOURCE_OBJ}
)
add_dependencies(${MODULE_NAME} generate_resource_object)
target_link_libraries(${MODULE_NAME} log riru::riru)


if (NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_custom_command(TARGET ${MODULE_NAME} POST_BUILD
            COMMAND ${CMAKE_STRIP} --strip-all --remove-section=.comment "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/lib${MODULE_NAME}.so")
endif ()
