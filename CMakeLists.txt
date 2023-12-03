# cmake arguments
# CMAKE_BUILD_TYPE:			Compilation target (Debug or Release defaults to Debug)
#
# godot-cpp cmake arguments
# GODOT_GDEXTENSION_DIR:		Path to the directory containing GDExtension interface header and API JSON file
# GODOT_CPP_SYSTEM_HEADERS		Mark the header files as SYSTEM. This may be useful to supress warnings in projects including this one.
# GODOT_CPP_WARNING_AS_ERROR	Treat any warnings as errors
# GODOT_CUSTOM_API_FILE:		Path to a custom GDExtension API JSON file (takes precedence over `gdextension_dir`)
# FLOAT_PRECISION:				Floating-point precision level ("single", "double")
#
# Android cmake arguments
# CMAKE_TOOLCHAIN_FILE:		The path to the android cmake toolchain ($ANDROID_NDK/build/cmake/android.toolchain.cmake)
# ANDROID_NDK:				The path to the android ndk root folder
# ANDROID_TOOLCHAIN_NAME:	The android toolchain (arm-linux-androideabi-4.9 or aarch64-linux-android-4.9 or x86-4.9 or x86_64-4.9)
# ANDROID_PLATFORM:			The android platform version (android-23)
# More info here: https://godot.readthedocs.io/en/latest/development/compiling/compiling_for_android.html
#
# Examples
#
# Builds a debug version:
# cmake .
# cmake --build .
#
# Builds a release version with clang
# CC=/usr/bin/clang CXX=/usr/bin/clang++ cmake -DCMAKE_BUILD_TYPE=Release -G "Unix Makefiles" .
# cmake --build .
#
# Builds an android armeabi-v7a debug version:
# cmake -DCMAKE_TOOLCHAIN_FILE=$ANDROID_NDK/build/cmake/android.toolchain.cmake -DANDROID_NDK=$ANDROID_NDK \
#		-DANDROID_TOOLCHAIN_NAME=arm-linux-androideabi-4.9 -DANDROID_PLATFORM=android-23 -DCMAKE_BUILD_TYPE=Debug .
# cmake --build .
#
# Protip
# Generate the buildfiles in a sub directory to not clutter the root directory with build files:
# mkdir build && cd build && cmake -G "Unix Makefiles" .. && cmake --build .
#
# Todo
# Test build for Windows, Mac and mingw.

cmake_minimum_required(VERSION 3.12)
project(godot-cpp LANGUAGES CXX)

option(GENERATE_TEMPLATE_GET_NODE "Generate a template version of the Node class's get_node." ON)
option(GODOT_CPP_SYSTEM_HEADERS "Expose headers as SYSTEM." ON)
option(GODOT_CPP_WARNING_AS_ERROR "Treat warnings as errors" OFF)

# Disable exception handling. Godot doesn't use exceptions anywhere, and this
# saves around 20% of binary size and very significant build time (GH-80513).
option(GODOT_DISABLE_EXCEPTIONS "Force disabling exception handling code" ON)

# Add path to modules
list( APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/" )

# Check if we are building ourself or being included
if(${PROJECT_NAME} STREQUAL ${CMAKE_PROJECT_NAME})
    set(GODOT_CPP_BUILDING_SELF ON)
endif()

# Set some helper variables for readability
set( compiler_is_clang "$<OR:$<CXX_COMPILER_ID:AppleClang>,$<CXX_COMPILER_ID:Clang>>" )
set( compiler_is_gnu "$<CXX_COMPILER_ID:GNU>" )
set( compiler_is_msvc "$<CXX_COMPILER_ID:MSVC>" )

# Default build type is Debug in the SConstruct
if("${CMAKE_BUILD_TYPE}" STREQUAL "")
	set(CMAKE_BUILD_TYPE Debug)
endif()

if(NOT DEFINED BITS)
	set(BITS 32)
	if(CMAKE_SIZEOF_VOID_P EQUAL 8)
		set(BITS 64)
	endif(CMAKE_SIZEOF_VOID_P EQUAL 8)
endif()

# Input from user for GDExtension interface header and the API JSON file
set(GODOT_GDEXTENSION_DIR "gdextension" CACHE STRING "")
set(GODOT_CUSTOM_API_FILE "" CACHE STRING "")

set(GODOT_GDEXTENSION_API_FILE "${GODOT_GDEXTENSION_DIR}/extension_api.json")
if (NOT "${GODOT_CUSTOM_API_FILE}" STREQUAL "")  # User-defined override.
	set(GODOT_GDEXTENSION_API_FILE "${GODOT_CUSTOM_API_FILE}")
endif()

set(FLOAT_PRECISION "single" CACHE STRING "")
if ("${FLOAT_PRECISION}" STREQUAL "double")
	add_definitions(-DREAL_T_IS_DOUBLE)
endif()

# Generate source from the bindings file
find_package(Python3 3.4 REQUIRED) # pathlib should be present
if(GENERATE_TEMPLATE_GET_NODE)
	set(GENERATE_BINDING_PARAMETERS "True")
else()
	set(GENERATE_BINDING_PARAMETERS "False")
endif()

execute_process(COMMAND "${Python3_EXECUTABLE}" "-c" "import binding_generator; binding_generator.print_file_list(\"${GODOT_GDEXTENSION_API_FILE}\", \"${CMAKE_CURRENT_BINARY_DIR}\", headers=True, sources=True)"
	WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
	OUTPUT_VARIABLE GENERATED_FILES_LIST
	OUTPUT_STRIP_TRAILING_WHITESPACE
)

add_custom_command(OUTPUT ${GENERATED_FILES_LIST}
		COMMAND "${Python3_EXECUTABLE}" "-c" "import binding_generator; binding_generator.generate_bindings(\"${GODOT_GDEXTENSION_API_FILE}\", \"${GENERATE_BINDING_PARAMETERS}\", \"${BITS}\", \"${FLOAT_PRECISION}\", \"${CMAKE_CURRENT_BINARY_DIR}\")"
		VERBATIM
		WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
		MAIN_DEPENDENCY ${GODOT_GDEXTENSION_API_FILE}
		DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/binding_generator.py
		COMMENT "Generating bindings"
)

# Get Sources
file(GLOB_RECURSE SOURCES CONFIGURE_DEPENDS src/*.c**)
file(GLOB_RECURSE HEADERS CONFIGURE_DEPENDS include/*.h**)

# Define our godot-cpp library
add_library(${PROJECT_NAME} STATIC
		${SOURCES}
		${HEADERS}
		${GENERATED_FILES_LIST}
)
add_library(godot::cpp ALIAS ${PROJECT_NAME})

if(MSVC)
	foreach(flag_var CMAKE_CXX_FLAGS CMAKE_CXX_FLAGS_DEBUG CMAKE_CXX_FLAGS_RELEASE)
		string(REPLACE "/EHsc" "" ${flag_var} "${${flag_var}}")
		string(REGEX REPLACE "/RTC[^ ]*" "" ${flag_var} "${${flag_var}}")
	endforeach(flag_var)
endif()

if(MSVC)
	# using Visual Studio C++
	set_property(TARGET ${PROJECT_NAME} PROPERTY MSVC_DEBUG_INFORMATION_FORMAT $<IF:$<CONFIG:DEBUG>,ProgramDatabase,>)
	set_property(TARGET ${PROJECT_NAME} PROPERTY MSVC_RUNTIME_LIBRARY MultiThreaded$<$<CONFIG:DEBUG>:Debug>DLL)

	target_compile_options(${PROJECT_NAME} PRIVATE /utf-8)
	target_compile_options(${PROJECT_NAME} PRIVATE /MP)
	target_compile_options(${PROJECT_NAME} PRIVATE $<$<CONFIG:Debug>:/RTC1>)

	target_compile_definitions(${PROJECT_NAME} PRIVATE WIN32_LEAN_AND_MEAN)
	target_compile_definitions(${PROJECT_NAME} PRIVATE NOMINMAX)
	# target_compile_definitions(${PROJECT_NAME} PRIVATE _CRT_SECURE_NO_WARNINGS)
	# target_compile_definitions(${PROJECT_NAME} PRIVATE _CRT_NONSTDC_NO_DEPRECATE)
else()  # GCC/Clang
	target_compile_options(${PROJECT_NAME} PRIVATE $<$<CONFIG:DEBUG>:-fno-omit-frame-pointer -O0 -g>)
	target_compile_options(${PROJECT_NAME} PRIVATE $<$<CONFIG:RELEASE>:-O3>)
endif()

if(GODOT_DISABLE_EXCEPTIONS)
	if(MSVC)
		target_compile_definitions(${PROJECT_NAME} PRIVATE _HAS_EXCEPTIONS=0)
	else()
		target_compile_options(${PROJECT_NAME} PRIVATE -fno-exceptions)
	endif()
else()
	if(MSVC)
		target_compile_definitions(${PROJECT_NAME} PRIVATE _HAS_EXCEPTIONS=1)
		target_compile_options(${PROJECT_NAME} PRIVATE /EHsc)
	endif()
endif()

include(GodotCompilerWarnings)

# Treat warnings as errors if we are building ourself
if(GODOT_CPP_BUILDING_SELF)
    unset( GODOT_CPP_WARNING_AS_ERROR CACHE )
    set_warning_as_error()
endif()

target_compile_features(${PROJECT_NAME}
	PRIVATE
		cxx_std_17
)

target_compile_definitions(${PROJECT_NAME} PUBLIC
	$<$<CONFIG:Debug>:
		DEBUG_ENABLED
		DEBUG_METHODS_ENABLED
	>
	$<${compiler_is_msvc}:
		TYPED_METHOD_BIND
	>
)

target_link_options(${PROJECT_NAME} PRIVATE
	$<$<NOT:${compiler_is_msvc}>:
		-static-libgcc
		-static-libstdc++
		-Wl,-R,'$$ORIGIN'
	>
)

# Optionally mark headers as SYSTEM
set(GODOT_CPP_SYSTEM_HEADERS_ATTRIBUTE "")
if (GODOT_CPP_SYSTEM_HEADERS)
	set(GODOT_CPP_SYSTEM_HEADERS_ATTRIBUTE SYSTEM)
endif ()

target_include_directories(${PROJECT_NAME} ${GODOT_CPP_SYSTEM_HEADERS_ATTRIBUTE} PUBLIC
	include
	${CMAKE_CURRENT_BINARY_DIR}/gen/include
	${GODOT_GDEXTENSION_DIR}
)

# Create the correct name (godot.os.build_type.system_bits)
string(TOLOWER "${CMAKE_SYSTEM_NAME}" SYSTEM_NAME)

if(ANDROID)
	# Added the android abi after system name
	set(SYSTEM_NAME ${SYSTEM_NAME}.${ANDROID_ABI})

	# Android does not have the bits at the end if you look at the main godot repo build
	set(OUTPUT_NAME "godot-cpp.${SYSTEM_NAME}.$<LOWER_CASE:$<CONFIG>>")
else()
	set(OUTPUT_NAME "godot-cpp.${SYSTEM_NAME}.$<LOWER_CASE:$<CONFIG>>.${BITS}")
endif()

set_target_properties(${PROJECT_NAME}
	PROPERTIES
		CXX_EXTENSIONS OFF
		POSITION_INDEPENDENT_CODE ON
		ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/bin"
		LIBRARY_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/bin"
		RUNTIME_OUTPUT_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/bin"
		OUTPUT_NAME "${OUTPUT_NAME}"
)
