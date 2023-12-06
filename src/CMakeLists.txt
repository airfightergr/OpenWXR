# CDDL HEADER START
#
# This file and its contents are supplied under the terms of the
# Common Development and Distribution License ("CDDL"), version 1.0.
# You may only use this file in accordance with the terms of version
# 1.0 of the CDDL.
#
# A full copy of the text of the CDDL should have accompanied this
# source.  A copy of the CDDL is also available via the Internet at
# http://www.illumos.org/license/CDDL.
#
# CDDL HEADER END

# Copyright 2018 Saso Kiselkov. All rights reserved.

cmake_minimum_required(VERSION 3.12)

if(APPLE)
	set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64" CACHE STRING "" FORCE)
endif(APPLE)
project(openwxr)

if(NOT CMAKE_BUILD_TYPE)
	SET(CMAKE_BUILD_TYPE "Debug")
endif()

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_C_STANDARD 11)

SET(SRC
    atmo_xp11.c
    dbg_log.c
    fontmgr.c
    standalone.c
    wxr.c
    xplane.c)
SET(HDR
    atmo.h
    atmo_xp11.h
    dbg_log.h
    fontmgr.h
    standalone.h
    wxr.h
    xplane.h)

SET(ALL_SRC ${SRC} ${HDR})
LIST(SORT ALL_SRC)


add_library(openwxr SHARED ${ALL_SRC})

if(WIN32)
	set(PLAT_SHORT "win64")
	set(PLAT_LONG "win-64")
	set(PLAT_ONLY "win")
	set(PLUGIN_BIN_OUTDIR "win_x64")
elseif(APPLE)
	set(PLAT_SHORT "mac64")
	set(PLAT_LONG "mac-64")
	set(PLAT_ONLY "mac")
	set(PLUGIN_BIN_OUTDIR "mac_x64")
else()
	set(PLAT_SHORT "lin64")
	set(PLAT_LONG "linux-64")
	set(PLAT_ONLY "linux")
	set(PLUGIN_BIN_OUTDIR "lin_x64")
endif()

set(CMAKE_C_FLAGS_RELWITHDEBINFO "${CMAKE_C_FLAGS_RELEASE} -DDEBUG")
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -DDEBUG")

execute_process(COMMAND git rev-parse --short HEAD
	OUTPUT_VARIABLE PLUGIN_VERSION WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}")

if(${PLUGIN_VERSION} STREQUAL "") 
	set(PLUGIN_VERSION "LOCAL")
else()
	string(REGEX REPLACE "\n$" "" PLUGIN_VERSION "${PLUGIN_VERSION}")
endif()

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DPLUGIN_VERSION=\"\"${PLUGIN_VERSION}\"\"")

if(NOT DEFINED ${XPSDK})
	set(XPSDK "${CMAKE_SOURCE_DIR}/libraries/SDK")
endif()

if(NOT DEFINED ${LIBACFUTILS})
	set(LIBACFUTILS "${CMAKE_SOURCE_DIR}/libraries/libacfutils_redist")
endif()

if(NOT DEFINED ${OPENAL})
	set(OPENAL "${CMAKE_SOURCE_DIR}/libraries/OpenAL")
endif()

if(NOT DEFINED ${OPENGPWS})
	set(OPENGPWS "${CMAKE_SOURCE_DIR}/libraries/opengpws")
endif()

add_definitions(-DXPLM200=1 -DXPLM210=1 -DXPLM300=1 -DXPLM301=1 -DXPLM303=1 -DXPLM400=1 -DGLEW_BUILD=GLEW_STATIC)

include_directories(openwxr PUBLIC
    "${LIBACFUTILS}/include"
	"${LIBACFUTILS}/include/cglm"
	"${XPSDK}/CHeaders/XPLM"
	"${XPSDK}/CHeaders/Widgets"
	"${OPENAL}/include"
    "../api"
    "${OPENGPWS}/api")

if(WIN32)
	include_directories(openwxr PUBLIC "${LIBACFUTILS}/mingw64/include")
else()
	include_directories(openwxr PUBLIC "${LIBACFUTILS}/${PLAT_SHORT}/include")
endif()

#compiler flags
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c11 -W -Wall -Wextra -Werror -fvisibility=hidden -Wno-unused-local-typedefs\
	-Wunused-result -Wno-missing-field-initializers")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} \
	-DCHECK_RESULT_USED=\"__attribute__ ((warn_unused_result))\"")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_GNU_SOURCE -D_FILE_OFFSET_BITS=64")

if(APPLE)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DPCRE2_CODE_UNIT_WIDTH=8 -DLACF_GLEW_USE_NATIVE_TLS=0")
endif()


if(APPLE)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DAPL=1 -DIBM=0 -DLIN=0")
elseif(WIN32)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DAPL=0 -DIBM=1 -DLIN=0 -D_WIN32_WINNT=0x0600")
else()
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DAPL=0 -DIBM=0 -DLIN=1")
endif()

#libraries

# linking
# Careful: order matters in target_link_libraries. Dependencies must be
# ordered as first symbol callers and then as symbol providers (i.e.
# libpng16 depends on libz, so linking must be first: libpng, then libz).


if(WIN32)
	set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${LIBACFUTILS}/mingw64/lib")	
else()
	set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "${LIBACFUTILS}/${PLAT_SHORT}/lib")	
endif()

link_libraries(m)
if(APPLE)
	set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} ${XPSDK}/Libraries/Mac)

	FIND_LIBRARY(XPLM_LIBRARY XPLM)
	FIND_LIBRARY(XPWIDGETS_LIBRARY XPWidgets)
	FIND_LIBRARY(GLEW_LIBRARY GLEWmx)
	FIND_LIBRARY(OPENGL_LIBRARY OpenGL)
	FIND_LIBRARY(OPENAL_LIBRARY OpenAL)
	FIND_LIBRARY(LIBACFUTILS_LIBRARY acfutils)
	FIND_LIBRARY(OPUSFILE_LIBRARY opusfile)
	FIND_LIBRARY(OPUS_LIBRARY opus)
	FIND_LIBRARY(OGG_LIBRARY ogg)
	FIND_LIBRARY(FREETYPE_LIBRARY freetype)
	FIND_LIBRARY(PIXMAN_LIBRARY pixman-1)
	FIND_LIBRARY(CAIRO_LIBRARY cairo)
	FIND_LIBRARY(CRYPTO_LIBRARY crypto)
	FIND_LIBRARY(CURL_LIBRARY curl)
	FIND_LIBRARY(SSL_LIBRARY ssl)	
	FIND_LIBRARY(LZMA_LIBRARY lzma)
	FIND_LIBRARY(LIBPNG_LIBRARY png16)
	FIND_LIBRARY(ZLIB_LIBRARY z)
	target_link_libraries(openwxr
	    ${LIBACFUTILS_LIBRARY}
	    ${OPENAL_LIBRARY}
		${GLEW_LIBRARY}
	    ${OPENGL_LIBRARY}
	    ${XPLM_LIBRARY}
	    ${XPWIDGETS_LIBRARY}
	    ${OPUSFILE_LIBRARY}
	    ${OPUS_LIBRARY}
	    ${OGG_LIBRARY}
	    ${CAIRO_LIBRARY}
	    ${PIXMAN_LIBRARY}
	    ${FREETYPE_LIBRARY}
	    ${LIBPNG_LIBRARY}
	    ${LZMA_LIBRARY}
		${CURL_LIBRARY}
		${SSL_LIBRARY}
		${CRYPTO_LIBRARY}
	    ${ZLIB_LIBRARY}
	    )
elseif(WIN32)
	find_package(OpenGL REQUIRED)
	set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} ${XPSDK}/Libraries/Win)

	FIND_LIBRARY(XPLM_LIBRARY XPLM_64)
	FIND_LIBRARY(LIBACFUTILS_LIBRARY acfutils)
	FIND_LIBRARY(OPUSFILE_LIBRARY opusfile)
	FIND_LIBRARY(OPUS_LIBRARY opus)
	FIND_LIBRARY(OGG_LIBRARY ogg)
	FIND_LIBRARY(FREETYPE_LIBRARY freetype)
	FIND_LIBRARY(PIXMAN_LIBRARY pixman-1)
	FIND_LIBRARY(CAIRO_LIBRARY cairo)
	FIND_LIBRARY(LIBPNG_LIBRARY png16)
	FIND_LIBRARY(CRYPTO_LIBRARY crypto)
	FIND_LIBRARY(CURL_LIBRARY curl)
	FIND_LIBRARY(SSL_LIBRARY ssl)	
	FIND_LIBRARY(ZLIB_LIBRARY z)
	FIND_LIBRARY(GLEW_LIBRARY glew32mx)
	FIND_LIBRARY(LZMA_LIBRARY lzma)
	FIND_LIBRARY(CURL_LIBRARY curl)
	target_link_libraries(openwxr
		${LIBACFUTILS_LIBRARY}
		${XPLM_LIBRARY}
		${OPUSFILE_LIBRARY}
		${OPUS_LIBRARY}
		${OGG_LIBRARY}
		${CAIRO_LIBRARY}
		${PIXMAN_LIBRARY}
		${FREETYPE_LIBRARY}
		${LIBPNG_LIBRARY}
		${ZLIB_LIBRARY}
		${GLEW_LIBRARY}
		${LZMA_LIBRARY}
		${CURL_LIBRARY}
		${SSL_LIBRARY}
		${CRYPTO_LIBRARY}
		${OPENGL_LIBRARY}
		-lpsapi
		-ldbghelp
		-lcrypt32 
		-lws2_32 
		-lwinmm
		)
else()
	find_package(OpenGL REQUIRED)

	FIND_LIBRARY(LIBACFUTILS_LIBRARY acfutils)
	FIND_LIBRARY(OPUSFILE_LIBRARY opusfile)
	FIND_LIBRARY(OPUS_LIBRARY opus)
	FIND_LIBRARY(OGG_LIBRARY ogg)
	FIND_LIBRARY(FREETYPE_LIBRARY freetype)
	FIND_LIBRARY(PIXMAN_LIBRARY pixman-1)
	FIND_LIBRARY(CAIRO_LIBRARY cairo)
	FIND_LIBRARY(LIBPNG_LIBRARY png16)
	FIND_LIBRARY(CRYPTO_LIBRARY crypto)
	FIND_LIBRARY(SSL_LIBRARY ssl)	
	FIND_LIBRARY(ZLIB_LIBRARY z)
	FIND_LIBRARY(GLEW_LIBRARY GLEWmx)
	FIND_LIBRARY(LZMA_LIBRARY lzma)
	FIND_LIBRARY(CURL_LIBRARY curl)
	FIND_LIBRARY(PTHREAD_LIBRARY pthread)
	FIND_LIBRARY(LIBC_NONSHARED c_nonshared)
	target_link_libraries(openwxr
		${LIBACFUTILS_LIBRARY}
		${OPENGL_LIBRARY}
		${OPUSFILE_LIBRARY}
		${OPUS_LIBRARY}
		${OGG_LIBRARY}
		${CAIRO_LIBRARY}
		${PIXMAN_LIBRARY}
		${FREETYPE_LIBRARY}
		${LIBPNG_LIBRARY}
		${ZLIB_LIBRARY}
		${GLEW_LIBRARY}
		${LZMA_LIBRARY}
		${CURL_LIBRARY}
		${SSL_LIBRARY}
		${CRYPTO_LIBRARY}
		${PTHREAD_LIBRARY}
		${LIBC_NONSHARED}
		)
endif()

if(APPLE)
	#set_target_properties(openwxr PROPERTIES LINK_FLAGS "-static-libstdc++")	
else()
	set_target_properties(openwxr PROPERTIES LINK_FLAGS "-static-libgcc -static-libstdc++ -Wl,--exclude-libs,ALL")
	if(UNIX AND NOT APPLE)
		set_target_properties(openwxr PROPERTIES LINK_FLAGS "-nodefaultlibs")
	endif()
endif()

set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES PREFIX "")
set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES SUFFIX "")
set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES RUNTIME_OUTPUT_DIRECTORY
	"${CMAKE_SOURCE_DIR}/Deploy/${CMAKE_PROJECT_NAME}/${PLUGIN_BIN_OUTDIR}/$<0:>" )
set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES LIBRARY_OUTPUT_DIRECTORY
	"${CMAKE_SOURCE_DIR}/Deploy/${CMAKE_PROJECT_NAME}/${PLUGIN_BIN_OUTDIR}/$<0:>" )
set_target_properties(${CMAKE_PROJECT_NAME} PROPERTIES OUTPUT_NAME "${CMAKE_PROJECT_NAME}.xpl")
