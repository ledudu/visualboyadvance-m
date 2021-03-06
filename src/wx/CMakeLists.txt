#Do not use this file directly.  Always use the top level CMakeLists.txt file
# This build is much easier if we just do it here.
SET( CMAKE_CXX_FLAGS -std=gnu++11 )

# not yet implemented
SET(CAIRO_DEFAULT OFF)

option(ENABLE_CAIRO "Enable Cairo rendering for the wxWidgets port" ${CAIRO_DEFAULT})

if( WIN32 )
  # not yet implemented
  option( ENABLE_DIRECT3D "Enable Direct3D rendering for the wxWidgets port" ON )
  option( ENABLE_XAUDIO2 "Enable xaudio2 sound output for the wxWidgets port" ON )
endif( WIN32 )

option( ENABLE_OPENAL "Enable OpenAL for the wxWidgets port" ON )

IF(WIN32 AND MINGW)
    IF(NOT EXISTS "${CMAKE_SOURCE_DIR}/dependencies")
        MESSAGE(FATAL_ERROR "Please run: git submodule update --init --recursive")
    ENDIF()

    INCLUDE_DIRECTORIES("${CMAKE_SOURCE_DIR}/dependencies/mingw-xaudio/include")
ENDIF()

IF(APPLE)
    ADD_DEFINITIONS(-DwxMAC_USE_CORE_GRAPHICS)
ENDIF(APPLE)

if( NOT ENABLE_CAIRO )
    ADD_DEFINITIONS (-DNO_CAIRO)
endif( NOT ENABLE_CAIRO )

if( NOT ENABLE_XAUDIO2 )
    ADD_DEFINITIONS (-DNO_XAUDIO2)
endif( NOT ENABLE_XAUDIO2 )

if(NOT ENABLE_DIRECT3D)
   ADD_DEFINITIONS(-DNO_D3D)
endif(NOT ENABLE_DIRECT3D)

if(ENABLE_OPENAL)
    FIND_PACKAGE(OpenAL REQUIRED)
    INCLUDE_DIRECTORIES(${OPENAL_INCLUDE_DIR})
else(ENABLE_OPENAL)
    ADD_DEFINITIONS (-DNO_OAL)
endif(ENABLE_OPENAL)

IF(CMAKE_BUILD_TYPE STREQUAL "Debug")
    SET(wxWidgets_USE_DEBUG ON) # noop if wx is compiled with --disable-debug, like in Mac Homebrew atm

    # and if this is the case, we can't set debug level without link failing
    IF(NOT wxWidgets_DEFINITIONS MATCHES "-DwxDEBUG_LEVEL=0")
        ADD_DEFINITIONS(-DwxDEBUG_LEVEL=1)
    ENDIF()
ENDIF()

IF(APPLE)
    SET(wxWidgets_USE_STATIC ON)
ENDIF(APPLE)

SET(wxWidgets_USE_UNICODE ON)
# adv is for wxAboutBox
# xml, html is for xrc
SET( wxWidgets_USE_LIBS xrc xml html adv gl net core base )
#list(APPEND wxWidgets_CONFIG_OPTIONS --version=2.8)
FIND_PACKAGE ( wxWidgets REQUIRED )
#EXECUTE_PROCESS(COMMAND sh "${wxWidgets_CONFIG_EXECUTABLE}" --cxxflags)
INCLUDE( ${wxWidgets_USE_FILE} )
FIND_PACKAGE ( Gettext REQUIRED )
FIND_PROGRAM(XGETTEXT xgettext)
FIND_PROGRAM(MSGINIT msginit)
if(ENABLE_NLS AND (NOT XGETTEXT OR NOT MSGINIT))
   message(SEND_ERROR "Cannot find gettext ${XGETTEXT} ${MSGINIT}")
endif(ENABLE_NLS AND (NOT XGETTEXT OR NOT MSGINIT))
IF(ENABLE_CAIRO)
    FIND_PACKAGE ( PkgConfig REQUIRED )
    PKG_CHECK_MODULES(CAIRO REQUIRED cairo)
    include_directories(${CAIRO_INCLUDE_DIRS})
    IF(WIN32)
       # need gdiplus to extract hdc for cairo context
       SET(CAIRO_LIBRARIES ${CAIRO_LIBRARIES} -lgdiplus)
    ENDIF(WIN32)
    # SET(CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS} ${CAIRO_CFLAGS})
ELSE(ENABLE_CAIRO)
    ADD_DEFINITIONS (-DNO_CAIRO)
    SET(CAIRO_LIBRARIES )
ENDIF(ENABLE_CAIRO)
IF(WIN32 AND ENABLE_DIRECTX)
    FIND_PACKGE ( DirectX REQUIRED )
ENDIF(WIN32 AND ENABLE_DIRECTX)

# contrib widgets
include_directories(widgets)

# for out-of-tree builds, grab includes from both target and source dirs
include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

# wxrc does not support xrs files in -c output (> 10x compression)
# we do it using the bin2.c utility

ADD_EXECUTABLE(bin2c bin2c.c)
# don't want SDL_main here
SET_TARGET_PROPERTIES(bin2c PROPERTIES COMPILE_FLAGS "-Dmain=main")

ADD_CUSTOM_COMMAND(OUTPUT wxvbam.xrs
                   WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
                   COMMAND wxrc ${CMAKE_CURRENT_SOURCE_DIR}/xrc/*.xrc -o wxvbam.xrs
		   DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/xrc/*.xrc)
ADD_CUSTOM_COMMAND(OUTPUT builtin-xrc.h
                   COMMAND bin2c wxvbam.xrs builtin-xrc.h builtin_xrs
		   DEPENDS bin2c wxvbam.xrs)

# use a built-in vba-over.ini if no config file present
ADD_CUSTOM_COMMAND(OUTPUT builtin-over.h
                   COMMAND bin2c ${CMAKE_CURRENT_SOURCE_DIR}/../vba-over.ini builtin-over.h builtin_over
                   DEPENDS bin2c ../vba-over.ini)

# I don't like duplicating/triplicating code, so I only declare
# event handlers once, and copy them in other places they are needed
# all using portable cmake code
ADD_CUSTOM_COMMAND(OUTPUT cmdtab.cpp cmdhandlers.h cmd-evtable.h
                   COMMAND
		      ${CMAKE_COMMAND} -D OUTDIR=${CMAKE_CURRENT_BINARY_DIR} -P copy-events.cmake
		   WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
		   DEPENDS cmdevents.cpp)

#
# the following should be in the main file for consistency with
# other front ends, but can't due to cmake issues
# then again, the main file should be split up into separate dirs anyway
#

SET( SRC_WX
   wxvbam.cpp
   guiinit.cpp
   viewers.cpp
   gfxviewers.cpp
   cmdevents.cpp
   opts.cpp
   sys.cpp
   panel.cpp
   viewsupt.cpp
   widgets/keyedit.cpp
   widgets/joyedit.cpp
   widgets/sdljoy.cpp
   widgets/wxmisc.cpp
   # probably ought to be in common
   ../sdl/text.cpp
   # from external source with minor modifications
   widgets/checkedlistctrl.cpp
   # generated
   cmdtab.cpp
   # generated includes must be explicitly listed
   builtin-xrc.h
   builtin-over.h
   cmdhandlers.h
   cmd-evtable.h
   #Icon File
   xrc/vbam.xpm
)

IF(APPLE)
    SET(SRC_WX ${SRC_WX} macsupport.mm)
ENDIF(APPLE)

SET( HDR_WX
   wxvbam.h
   drawing.h
   filters.h
   ioregs.h
   opts.h
   viewsupt.h
   wxhead.h
   widgets/wx/keyedit.h
   widgets/wx/joyedit.h
   widgets/wx/sdljoy.h
   widgets/wx/webupdatedef.h
   widgets/wx/wxmisc.h
   # probably ought to be in common
   ../sdl/text.h
   # from external source with minor modifications
   widgets/wx/checkedlistctrl.h
)

FILE(GLOB XRC_WX xrc/*.xrc)

SET( RES_WX
    ${XRC_WX}
    ../vba-over.ini
)

SET( CM_STUFF
    copy-events.cmake
)

IF(ENABLE_OPENAL)
    SET( SRC_WX ${SRC_WX} openal.cpp )
    SET( HDR_WX ${HDR_WX} openal.h )
ENDIF(ENABLE_OPENAL)

IF(ENABLE_XAUDIO2)
	SET( SRC_WX ${SRC_WX} xaudio2.cpp )
ENDIF(ENABLE_XAUDIO2)

IF( WIN32 )
    SET( SRC_WX ${SRC_WX} wxvbam.rc dsound.cpp )
    SET(DIRECTX_LIBRARIES -ldxguid -ldsound -lws2_32)
    # not strictly directx, but win32-related
    IF(ENABLE_DEBUGGER)
        SET(DIRECTX_LIBRARIES ${DIRECTX_LIBRARIES} -lwsock32)
    ENDIF(ENABLE_DEBUGGER)
ENDIF( WIN32 )

link_directories( ${CMAKE_BINARY_DIR} )

SET(VBAM_ICON vbam.icns)

SET(VBAM_ICON_PATH ${CMAKE_CURRENT_SOURCE_DIR}/icons/${VBAM_ICON})

IF(APPLE)
    SET(CMAKE_BUILD_WITH_INSTALL_RPATH ON)
    SET(CMAKE_INSTALL_RPATH "@loader_path/../Frameworks")
ENDIF(APPLE)

ADD_EXECUTABLE (
    visualboyadvance-m
    WIN32
    MACOSX_BUNDLE
    ${SRC_WX}
    ${HDR_WX}
    ${RES_WX}
    ${VBAM_ICON_PATH}
    ${CM_STUFF}
)

TARGET_LINK_LIBRARIES (
    visualboyadvance-m
    ${VBAMCORE_LIBS}
    ${wxWidgets_LIBRARIES}
    ${FFMPEG_LIBRARIES}
    ${DIRECTX_LIBRARIES}
    ${CAIRO_LIBRARIES}
)
IF(WIN32)
    INSTALL(PROGRAMS ${PROJECT_BINARY_DIR}/visualboyadvance-m${CMAKE_EXECUTABLE_SUFFIX} DESTINATION ${CMAKE_BINARY_DIR})
ENDIF(WIN32)
IF(NOT WIN32 AND NOT APPLE)
    INSTALL(FILES ${CMAKE_CURRENT_SOURCE_DIR}/wxvbam.desktop DESTINATION share/applications)
ENDIF(NOT WIN32 AND NOT APPLE)

if(APPLE)
    # this should set ROM file types correctly
    SET_PROPERTY(TARGET visualboyadvance-m APPEND PROPERTY MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/wxplist.in)
    SET(MACOSX_BUNDLE_ICON_FILE ${VBAM_ICON})
    SET_SOURCE_FILES_PROPERTIES(${VBAM_ICON_PATH} PROPERTIES MACOSX_PACKAGE_LOCATION Resources)

    # budle dylibs and relink them for releasing .app
    # but only in Release mode
    IF(NOT CMAKE_BUILD_TYPE STREQUAL "Debug")
        ADD_CUSTOM_COMMAND(TARGET visualboyadvance-m POST_BUILD
            COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/tools/osx/third_party_libs_tool "$<TARGET_FILE_DIR:visualboyadvance-m>/../..")
    ENDIF()
endif(APPLE)

SET(WX_EXE_NAME visualboyadvance-m-wx${CMAKE_EXECUTABLE_SUFFIX})
