cmake_minimum_required(VERSION 3.11)

set(BC7ENC_VERSION 1.0.0)

project(Bc7Enc VERSION ${BC7ENC_VERSION} LANGUAGES C CXX)

option(BUILD_X64 "build 64-bit" TRUE)
option(SUPPORT_BC7E "support BC7E (requires ispc)" FALSE)

message("Initial BUILD_X64=${BUILD_X64}")
message("Initial CMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
message("Initial SUPPORT_BC7E=${SUPPORT_BC7E}")

if( NOT CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release )
endif( NOT CMAKE_BUILD_TYPE )

message( ${PROJECT_NAME} " build type: " ${CMAKE_BUILD_TYPE} )

if (BUILD_X64)
	message("Building 64-bit")
else()
	message("Building 32-bit")
endif(BUILD_X64)

if (NOT MSVC)
	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -fopenmp")
	set(CMAKE_C_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -fopenmp")

	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fopenmp")
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -fopenmp")
else()
	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /openmp /W4")
	set(CMAKE_C_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /openmp /W4")

	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /openmp /W4")
	set(CMAKE_C_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /openmp /W4")
endif()

if (SUPPORT_BC7E)
    find_program(ISPC_EXECUTABLE ispc)
    if(NOT ISPC_EXECUTABLE)
        message(FATAL_ERROR "SUPPORT_BC7E is ON but 'ispc' was not found in PATH")
    endif()

	if (NOT MSVC)
		add_custom_command(OUTPUT ${CMAKE_SOURCE_DIR}/bc7e.o ${CMAKE_SOURCE_DIR}/bc7e_avx.o ${CMAKE_SOURCE_DIR}/bc7e_sse2.o ${CMAKE_SOURCE_DIR}/bc7e_sse4.o ${CMAKE_SOURCE_DIR}/bc7e_avx2.o
			COMMAND ${ISPC_EXECUTABLE} -g -O2 ${CMAKE_SOURCE_DIR}/bc7e.ispc -o ${CMAKE_SOURCE_DIR}/bc7e.o -h ${CMAKE_SOURCE_DIR}/include/bc7e_ispc.h --target=sse2,sse4,avx,avx2 --opt=fast-math --opt=disable-assertions
										 	DEPENDS bc7e.ispc)
	else()
		add_custom_command(OUTPUT ${CMAKE_SOURCE_DIR}/bc7e.obj ${CMAKE_SOURCE_DIR}/bc7e_avx.obj ${CMAKE_SOURCE_DIR}/bc7e_sse2.obj ${CMAKE_SOURCE_DIR}/bc7e_sse4.obj ${CMAKE_SOURCE_DIR}/bc7e_avx2.obj
			COMMAND ${ISPC_EXECUTABLE} -g -O2 ${CMAKE_SOURCE_DIR}/bc7e.ispc -o ${CMAKE_SOURCE_DIR}/bc7e.obj -h ${CMAKE_SOURCE_DIR}/include/bc7e_ispc.h --target=sse2,sse4,avx,avx2 --opt=fast-math --opt=disable-assertions
										 	DEPENDS bc7e.ispc)
	endif()
endif()

# -fno-strict-aliasing shouldn't be necessary, it's here because that is what MSVC uses by default and that's what I've tested with the most.
if (NOT MSVC)
	set(GCC_COMPILE_FLAGS "-fno-strict-aliasing -Wall -Wextra")
	if (NOT BUILD_X64)
		set(GCC_COMPILE_FLAGS "${GCC_COMPILE_FLAGS} -m32")
	endif()
endif()	

set(CMAKE_C_FLAGS  "${CMAKE_C_FLAGS} ${GCC_COMPILE_FLAGS}")
set(CMAKE_C_FLAGS_RELEASE  "${CMAKE_C_FLAGS_RELEASE} ${GCC_COMPILE_FLAGS}")
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${GCC_COMPILE_FLAGS} -D_DEBUG")

if (SUPPORT_BC7E)
	set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} ${GCC_COMPILE_FLAGS} -DSUPPORT_BC7E=1")
else()
	set(CMAKE_CXX_FLAGS  "${CMAKE_CXX_FLAGS} ${GCC_COMPILE_FLAGS}")
endif()

set(CMAKE_CXX_FLAGS_RELEASE  "${CMAKE_CXX_FLAGS_RELEASE} ${GCC_COMPILE_FLAGS}")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} ${GCC_COMPILE_FLAGS} -D_DEBUG")

set(BC7ENC_SRC_LIST  ${COMMON_SRC_LIST}
    "${CMAKE_CURRENT_SOURCE_DIR}/src/bc7enc.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/bc7decomp.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/bc7decomp_ref.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/lodepng.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/rgbcx.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/utils.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/ert.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/rdo_bc_encoder.cpp"
)

set(BC7ENC_HEADER_LIST
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/bc7enc.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/bc7decomp.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/lodepng.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/rgbcx.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/rgbcx_table4.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/rgbcx_table4_small.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/utils.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/miniz.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/dds_defs.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/ert.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/include/bc7enc/rdo_bc_encoder.h"
)

if (NOT SUPPORT_BC7E)
		add_library(Bc7Enc ${BC7ENC_SRC_LIST} ${BC7ENC_HEADER_LIST})
else()
	if (NOT MSVC)
		add_library(Bc7Enc ${BC7ENC_SRC_LIST} ${BC7ENC_HEADER_LIST} ${CMAKE_SOURCE_DIR}/bc7e.o ${CMAKE_SOURCE_DIR}/bc7e_avx.o ${CMAKE_SOURCE_DIR}/bc7e_avx2.o ${CMAKE_SOURCE_DIR}/bc7e_sse2.o ${CMAKE_SOURCE_DIR}/bc7e_sse4.o)
	else()
		add_library(Bc7Enc ${BC7ENC_SRC_LIST} ${BC7ENC_HEADER_LIST} ${CMAKE_SOURCE_DIR}/bc7e.obj ${CMAKE_SOURCE_DIR}/bc7e_avx.obj ${CMAKE_SOURCE_DIR}/bc7e_avx2.obj ${CMAKE_SOURCE_DIR}/bc7e_sse2.obj ${CMAKE_SOURCE_DIR}/bc7e_sse4.obj)
	endif()
endif()

if (NOT MSVC)
	target_link_libraries(Bc7Enc m)
endif()

set_target_properties(Bc7Enc PROPERTIES
    PUBLIC_HEADER "${BC7ENC_HEADER_LIST}"
)

target_include_directories(Bc7Enc PUBLIC
	$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
	$<INSTALL_INTERFACE:include>
)

include(GNUInstallDirs)

install(TARGETS Bc7Enc
        EXPORT Bc7EncTargets
        ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
        RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
		INCLUDES DESTINATION include
        PUBLIC_HEADER DESTINATION include/bc7enc)

install(EXPORT Bc7EncTargets
    FILE ${PROJECT_NAME}Targets.cmake
	NAMESPACE ${PROJECT_NAME}::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

set(CONFIG_FILE_INSTALL_DIR "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
include(CMakePackageConfigHelpers)

configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
    INSTALL_DESTINATION ${CONFIG_FILE_INSTALL_DIR}
)

write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY AnyNewerVersion
)

install(FILES
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)
