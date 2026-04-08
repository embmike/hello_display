cmake_minimum_required(VERSION 3.15)

project(hello_display LANGUAGES C CXX ASM)

find_bsp(ID com.sysprogs.project_importers.stm32.cubemx SOURCE_PROJECT hello_display.ioc)

add_bsp_based_executable(NAME hello_display
	GENERATE_BIN
	GENERATE_MAP)

