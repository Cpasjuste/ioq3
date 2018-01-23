cmake_minimum_required(VERSION 2.8)

if (NOT DEFINED CMAKE_TOOLCHAIN_FILE)
    if (DEFINED ENV{VITASDK})
        set(CMAKE_TOOLCHAIN_FILE "$ENV{VITASDK}/share/vita.toolchain.cmake" CACHE PATH "toolchain file")
    else ()
        message(FATAL_ERROR "Please define VITASDK to point to your SDK path!")
    endif ()
endif ()

project(ioq3)
include("${VITASDK}/share/vita.cmake" REQUIRED)

set(VITA_APP_NAME "Quake 3")
set(VITA_TITLEID "IOQ300001")
set(VITA_VERSION "01.00")

enable_language(ASM)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
set(VITA_MKSFOEX_FLAGS "${VITA_MKSFOEX_FLAGS} -d PARENTAL_LEVEL=1")

set(SRC ${CMAKE_CURRENT_SOURCE_DIR})

set(INCLUDES
        ${SRC}/code/client
        ${SRC}/code/qcommon
        ${SRC}/code/server
        ${SRC}/code/botlib
        ${SRC}/code/sys
        ${SRC}/code/renderercommon
        ${SRC}/code/renderergl1
        ${SRC}/code/sdl
        ${SRC}/code/psp2
        ${VITASDK}/arm-vita-eabi/include/SDL2
        )

set(FLAGS
        -D__PSP2__ -D__FLOAT_WORD_ORDER=1 -D__GNU__
        -DUSE_ICON -DARCH_STRING="arm"
        -DNO_VM_COMPILED -DDEFAULT_BASEDIR=\"ux0:/data/ioq3\"
        -DPRODUCT_VERSION=\"1.36_GIT_ba68b99c-2018-01-23\"
        -mfpu=neon -mcpu=cortex-a9 -march=armv7-a -mfloat-abi=hard -ffast-math
        -fno-asynchronous-unwind-tables -funroll-loops
        -mword-relocations -fno-unwind-tables -fno-optimize-sibling-calls
        -mvectorize-with-neon-quad -funsafe-math-optimizations
        -mlittle-endian -munaligned-access
        -fsingle-precision-constant
        -fno-strict-aliasing -Wimplicit -Wstrict-prototypes -pipe
        -Wformat=2 -Wno-format-zero-length -Wformat-security -Wno-format-nonliteral -Wstrict-aliasing=2
        -Wmissing-format-attribute -Wdisabled-optimization -Werror-implicit-function-declaration
        -Wno-unused-variable -Wno-unused-but-set-variable
        #-fno-short-enums
        )

##############
## BOTLIB
##############
file(GLOB BOTLIB_SRC ${SRC}/code/botlib/*.c)
add_library(qbot ${BOTLIB_SRC})
target_compile_options(qbot PRIVATE ${FLAGS} -DBOTLIB)
target_include_directories(qbot PRIVATE ${INCLUDES})

##############
## CLIENT
##############
file(GLOB CLIENT_SRC ${SRC}/code/client/*.c)
list(REMOVE_ITEM CLIENT_SRC ${SRC}/code/client/libmumblelink.c)
add_library(qclient ${CLIENT_SRC})
target_compile_options(qclient PRIVATE ${FLAGS} -DUSE_CODEC_VORBIS)
target_include_directories(qclient PRIVATE ${INCLUDES})

####################
## SERVER
####################
file(GLOB SERVER_SRC ${SRC}/code/server/*.c)
list(REMOVE_ITEM SERVER_SRC ${SRC}/code/server/sv_rankings.c)
add_library(qserver ${SERVER_SRC})
target_compile_options(qserver PRIVATE ${FLAGS} -DUSE_CODEC_VORBIS)
target_include_directories(qserver PRIVATE ${INCLUDES})

##############
## QCOMMON
##############
file(GLOB QCOMMON_SRC ${SRC}/code/qcommon/*.c)
list(REMOVE_ITEM QCOMMON_SRC
        ${SRC}/code/qcommon/net_ip.c
        ${SRC}/code/qcommon/vm_armv7l.c
        ${SRC}/code/qcommon/vm_none.c
        ${SRC}/code/qcommon/vm_powerpc.c
        ${SRC}/code/qcommon/vm_powerpc_asm.c
        ${SRC}/code/qcommon/vm_sparc.c
        ${SRC}/code/qcommon/vm_x86.c
        )
add_library(qcommon ${QCOMMON_SRC})
target_compile_options(qcommon PRIVATE ${FLAGS} -DUSE_CODEC_VORBIS)
target_include_directories(qcommon PRIVATE ${INCLUDES})

####################
## RENDERER COMMON
####################
file(GLOB RENDERER_COMMON_SRC ${SRC}/code/renderercommon/*.c)
add_library(qrenderercommon ${RENDERER_COMMON_SRC})
target_compile_options(qrenderercommon PRIVATE ${FLAGS} -DUSE_CODEC_VORBIS)
target_include_directories(qrenderercommon PRIVATE ${INCLUDES})

####################
## RENDERER GL1
####################
file(GLOB RENDERER_GL1_SRC ${SRC}/code/renderergl1/*.c)
list(REMOVE_ITEM RENDERER_GL1_SRC ${SRC}/code/renderergl1/tr_subs.c)
add_library(qrenderergl1 ${RENDERER_GL1_SRC})
target_compile_options(qrenderergl1 PRIVATE ${FLAGS} -DUSE_CODEC_VORBIS)
target_include_directories(qrenderergl1 PRIVATE ${INCLUDES})

####################
## SDL
####################
file(GLOB SDL_SRC ${SRC}/code/sdl/*.c)
add_library(qsdl ${SDL_SRC})
target_compile_options(qsdl PRIVATE ${FLAGS} -DUSE_CODEC_VORBIS)
target_include_directories(qsdl PRIVATE ${INCLUDES})

####################
####################
##
## ioq3 !
##
####################
####################

add_executable(${PROJECT_NAME}
        ${SRC}/code/sys/con_log.c
        ${SRC}/code/psp2/sys_main.c
        ${SRC}/code/psp2/sys_psp2.c
        ${SRC}/code/psp2/con_psp2.c
        ${SRC}/code/psp2/net_psp2.c
        )
target_include_directories(${PROJECT_NAME} PRIVATE ${INCLUDES})
target_compile_options(${PROJECT_NAME} PRIVATE ${FLAGS})
target_link_libraries(${PROJECT_NAME} PRIVATE
        qclient qserver qbot qrenderergl1 qrenderercommon qcommon qsdl
        SDL2 vita2d
        SceAppMgr_stub SceNet_stub SceNetCtl_stub
        SceDisplay_stub SceGxm_stub SceSysmodule_stub SceAudio_stub
        SceCommonDialog_stub SceCtrl_stub SceTouch_stub SceHid_stub ScePower_stub
        vorbisfile vorbis ogg jpeg z
        OSMesa ssl crypto pthread m
        )

vita_create_self(${PROJECT_NAME}.self ${PROJECT_NAME})
vita_create_vpk(${PROJECT_NAME}.vpk ${VITA_TITLEID} ${PROJECT_NAME}.self
        VERSION ${VITA_VERSION}
        NAME ${VITA_APP_NAME}
        )
