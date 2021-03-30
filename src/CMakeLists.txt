include(CheckCXXCompilerFlag)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules")
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

if(UNIX OR MINGW)
	add_definitions(-D_FILE_OFFSET_BITS=64)
endif()

CHECK_CXX_COMPILER_FLAG(-pthread CXX_HAS_PTHREAD)
if(CXX_HAS_PTHREAD)
    add_definitions(-pthread)
endif()

CHECK_CXX_COMPILER_FLAG(-ffast-math CXX_HAS_FFAST_MATH)
if(CXX_HAS_FFAST_MATH)
    add_definitions(-ffast-math)
endif()

find_library(LIBM m REQUIRED)
find_library(LIBDL dl REQUIRED)
find_library(LIBPTHREAD pthread REQUIRED)

find_package(PkgConfig REQUIRED)

pkg_check_modules(CONFIG REQUIRED libconfig++)
list(APPEND rtl_airband_extra_libs ${CONFIG_LIBRARIES})
list(APPEND rtl_airband_include_dirs ${CONFIG_INCLUDE_DIRS})
list(APPEND link_dirs ${CONFIG_LIBRARY_DIRS})

# Can't use pkg_check_modules here, as some distros do not install lame.pc file
find_package(Lame REQUIRED)
list(APPEND rtl_airband_extra_libs ${LAME_LIBRARIES})
list(APPEND rtl_airband_include_dirs ${LAME_INCLUDE_DIR})

pkg_check_modules(SHOUT REQUIRED shout)
list(APPEND rtl_airband_extra_libs ${SHOUT_LIBRARIES})
list(APPEND rtl_airband_include_dirs ${SHOUT_INCLUDE_DIRS})
list(APPEND link_dirs ${SHOUT_LIBRARY_DIRS})

option(NFM "Enable support for narrow FM channels" OFF)

set(PLATFORM "native" CACHE STRING "Optimize the build for the given hardware platform")

option(RTLSDR "Enable RTL-SDR support" ON)
set(WITH_RTLSDR FALSE)

option(MIRISDR "Enable Mirics support" ON)
set(WITH_MIRISDR FALSE)

option(SOAPYSDR "Enable SoapySDR support" ON)
set(WITH_SOAPYSDR FALSE)

option(PULSEAUDIO "Enable PulseAudio support" ON)
set(WITH_PULSEAUDIO FALSE)

option(PROFILING "Enable profiling with gperftools")
set(WITH_PROFILING FALSE)

if(RTLSDR)
	find_package(RTLSDR)
	if(RTLSDR_FOUND)
		list(APPEND rtl_airband_extra_sources input-rtlsdr.cpp)
		list(APPEND rtl_airband_extra_libs ${RTLSDR_LIBRARIES})
		list(APPEND rtl_airband_include_dirs ${RTLSDR_INCLUDE_DIRS})
		list(APPEND link_dirs ${RTLSDR_LIBRARY_DIRS})
		set(WITH_RTLSDR TRUE)
	endif()
endif()

if(MIRISDR)
	find_package(MiriSDR)
	if(MIRISDR_FOUND)
		set(WITH_MIRISDR TRUE)
		list(APPEND rtl_airband_extra_sources input-mirisdr.cpp)
		list(APPEND rtl_airband_extra_libs ${MIRISDR_LIBRARIES})
		list(APPEND rtl_airband_include_dirs ${MIRISDR_INCLUDE_DIRS})
		list(APPEND link_dirs ${MIRISDR_LIBRARY_DIRS})
	endif()
endif()

if(SOAPYSDR)
	message(STATUS "Checking for SoapySDR")
	find_package(SoapySDR NO_MODULE)
	if(SoapySDR_FOUND)
		list(APPEND rtl_airband_extra_sources input-soapysdr.cpp)
		message(STATUS "  SoapySDR found, ${SoapySDR_INCLUDE_DIRS}, ${SoapySDR_LIBRARIES}")
		list(APPEND rtl_airband_extra_libs ${SoapySDR_LIBRARIES})
		list(APPEND rtl_airband_include_dirs ${SoapySDR_INCLUDE_DIRS})
		set(WITH_SOAPYSDR TRUE)
	else()
		message(STATUS "  SoapySDR not found")
	endif()
endif()

if(PULSEAUDIO)
	pkg_check_modules(PULSEAUDIO libpulse)
	if(PULSEAUDIO_FOUND)
		list(APPEND rtl_airband_extra_sources pulse.cpp)
		list(APPEND rtl_airband_extra_libs ${PULSEAUDIO_LIBRARIES})
		list(APPEND rtl_airband_include_dirs ${PULSEAUDIO_INCLUDE_DIRS})
		list(APPEND link_dirs ${PULSEAUDIO_LIBRARY_DIRS})
		set(WITH_PULSEAUDIO TRUE)
	endif()
endif()

if(PROFILING)
	pkg_check_modules(PROFILING libprofiler)
	if(PROFILING_FOUND)
		list(APPEND rtl_airband_extra_libs ${PROFILING_LIBRARIES})
		list(APPEND rtl_airband_include_dirs ${PROFILING_INCLUDE_DIRS})
		list(APPEND link_dirs ${PROFILING_LIBRARY_DIRS})
		set(WITH_PROFILING TRUE)
	endif()
endif()


option(BCM_VC "Enable Broadcom Videocore 3 support" OFF)
set(USE_BCM_VC FALSE)

if(PLATFORM STREQUAL "rpiv1")
	set(BCM_VC ON)
	add_definitions(-march=armv6zk -mcpu=arm1176jzf-s -mtune=arm1176jzf-s -mfpu=vfp)
	enable_language(ASM)
	list(APPEND rtl_airband_extra_sources rtl_airband_vfp.s)
elseif(PLATFORM STREQUAL "rpiv2")
	set(BCM_VC ON)
	add_definitions(-march=armv7-a -mfpu=neon-vfpv4 -mfloat-abi=hard)
	enable_language(ASM)
	list(APPEND rtl_airband_extra_sources rtl_airband_neon.s)
elseif(PLATFORM STREQUAL "armv7-generic")
	add_definitions(-march=armv7-a -mfpu=neon-vfpv4 -mfloat-abi=hard)
elseif(PLATFORM STREQUAL "armv8-generic")
	add_definitions(-march=armv8-a+crc -mtune=cortex-a53)
elseif(PLATFORM STREQUAL "native")
	add_definitions(-march=native -mtune=native)
else()
	message(FATAL_ERROR "Unknown platform '${PLATFORM}'. Valid options are:
	rpiv1, rpiv2, armv7-generic, armv8-generic, native")
endif()

if(BCM_VC)
	find_package(BCM_VC)
	if(BCM_VC_FOUND)
		list(APPEND rtl_airband_extra_libs ${BCM_VC_LIBRARIES} hello_fft)
		add_subdirectory(hello_fft)
		set(USE_BCM_VC TRUE)
	endif()
else()
	pkg_check_modules(FFTW3F REQUIRED fftw3f)
	if(FFTW3F_FOUND)
		list(APPEND rtl_airband_extra_libs ${FFTW3F_LIBRARIES})
		list(APPEND rtl_airband_include_dirs ${FFTW3F_INCLUDE_DIRS})
		list(APPEND link_dirs ${FFTW3F_LIBRARY_DIRS})
	endif()
endif()

if(CMAKE_SYSTEM_NAME STREQUAL "FreeBSD")
	list(APPEND rtl_airband_extra_libs c++)
endif()

message(STATUS "RTLSDR-Airband configuration summary:\n")
message(STATUS "- Build type:\t\t${CMAKE_BUILD_TYPE}")
message(STATUS "- Operating system:\t\t${CMAKE_SYSTEM_NAME}")
message(STATUS "- SDR drivers:")
message(STATUS "  - librtlsdr:\t\trequested: ${RTLSDR}, enabled: ${WITH_RTLSDR}")
message(STATUS "  - mirisdr:\t\t\trequested: ${MIRISDR}, enabled: ${WITH_MIRISDR}")
message(STATUS "  - soapysdr:\t\trequested: ${SOAPYSDR}, enabled: ${WITH_SOAPYSDR}")
message(STATUS "- Other options:")
message(STATUS "  - Platform:\t\t${PLATFORM}")
message(STATUS "  - Broadcom VideoCore GPU:\t${USE_BCM_VC}")
message(STATUS "  - NFM support:\t\t${NFM}")
message(STATUS "  - PulseAudio:\t\trequested: ${PULSEAUDIO}, enabled: ${WITH_PULSEAUDIO}")
message(STATUS "  - Profiling:\t\trequested: ${PROFILING}, enabled: ${WITH_PROFILING}")

configure_file(
	"${CMAKE_CURRENT_SOURCE_DIR}/config.h.in"
	"${CMAKE_CURRENT_SOURCE_DIR}/config.h"
	@ONLY
)

add_custom_command(
	OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/version.cpp
		${CMAKE_CURRENT_BINARY_DIR}/_version.cpp
	COMMAND ${CMAKE_COMMAND} -DRTL_AIRBAND_VERSION=${RTL_AIRBAND_VERSION} -P
	${CMAKE_CURRENT_SOURCE_DIR}/CMakeModules/version.cmake
)

add_library (rtl_airband_base OBJECT
	config.cpp
	input-common.cpp
	input-file.cpp
	input-helpers.cpp
	mixer.cpp
	output.cpp
	rtl_airband.cpp
	squelch.cpp
	util.cpp
	${CMAKE_CURRENT_BINARY_DIR}/version.cpp
	${rtl_airband_extra_sources}
	)

target_include_directories (rtl_airband_base PUBLIC
	${rtl_airband_include_dirs}
)

# can't do this per target with cmake <3.13
link_directories(${link_dirs})

set(rtl_airband_obj_files
	$<TARGET_OBJECTS:rtl_airband_base>
)

add_executable (rtl_airband ${rtl_airband_obj_files})

target_link_libraries (rtl_airband
	dl
	m
	pthread
	${rtl_airband_extra_libs}
)

install(TARGETS rtl_airband
	RUNTIME DESTINATION bin
)

# TODO: install config if not present