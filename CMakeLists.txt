
cmake_minimum_required(VERSION 3.14)

# initialize the SDK based on PICO_SDK_PATH
include($ENV{PICO_SDK_PATH}/external/pico_sdk_import.cmake)

project(Custom-Bootloader)
set(CMAKE_C_STANDARD 11)

# initialize the Raspberry Pi Pico SDK
pico_sdk_init()

add_executable(test test.c)

# add pico stdlib library which aggregates commonly used features
target_link_libraries(test pico_stdlib)

# extra settings
pico_enable_stdio_usb(test 1)
pico_enable_stdio_uart(test 0)

# create map/bin/hex/uf2 files in addition to ELF
pico_add_extra_outputs(test)


