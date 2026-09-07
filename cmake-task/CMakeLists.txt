cmake_minimum_required(VERSION 3.16)

project(calculator LANGUAGES C)
add_executable(calculator
    src/main.c
    src/calculator.c
    src/logger.c
)

target_include_directories(calculator
    PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/include
)
