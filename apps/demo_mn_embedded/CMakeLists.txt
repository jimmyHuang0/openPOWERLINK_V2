################################################################################
#
# CMake file of CiA302 MN embedded demo application
#
# Copyright (c) 2016, Bernecker+Rainer Industrie-Elektronik Ges.m.b.H. (B&R)
# Copyright (c) 2016, Kalycito Infotech Private Limited
# All rights reserved.
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#     * Redistributions of source code must retain the above copyright
#       notice, this list of conditions and the following disclaimer.
#     * Redistributions in binary form must reproduce the above copyright
#       notice, this list of conditions and the following disclaimer in the
#       documentation and/or other materials provided with the distribution.
#     * Neither the name of the copyright holders nor the
#       names of its contributors may be used to endorse or promote products
#       derived from this software without specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND
# ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED
# WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
# DISCLAIMED. IN NO EVENT SHALL COPYRIGHT HOLDERS BE LIABLE FOR ANY
# DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES
# (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES;
# LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND
# ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS
# SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
################################################################################

################################################################################
# Setup project and generic options

PROJECT(demo_mn_embedded C)
MESSAGE(STATUS "Configuring ${PROJECT_NAME}")

CMAKE_MINIMUM_REQUIRED(VERSION 2.8.7)

# Set CMake Policy to suppress the warnings in CMake version 3.3.x
IF ((POLICY CMP0043) OR (POLICY CMP0053))
    CMAKE_POLICY(SET CMP0043 OLD)
    CMAKE_POLICY(SET CMP0053 OLD)
ENDIF()

INCLUDE(../common/cmake/options.cmake)

################################################################################
# Setup project files and definitions

SET(OBJDICT CiA302-4_MN)              # Set object dictionary to use
FIND_OPLK_LIBRARY("mn")

OPTION(CFG_IP_STACK "Is an IP stack available (support of SDO/UDP)" OFF)

IF (CFG_FAT_SUPPORT)
    SET(CFG_CDC_ON_SD ON CACHE BOOL "CDC on external SD card; to compile the cdc with application uncheck this option.")
ELSE()
    UNSET(CFG_CDC_ON_SD)
ENDIF()

# Set generic sources and include directories for this demo
SET(DEMO_SOURCES
    ${DEMO_SOURCE_DIR}/main.c
    ${DEMO_SOURCE_DIR}/app.c
    ${DEMO_SOURCE_DIR}/event.c
    ${COMMON_SOURCE_DIR}/obdcreate/obdcreate.c
    ${OBJDICT_DIR}/${OBJDICT}/obdpi.c
    ${COMMON_SOURCE_DIR}/lcd/lcd.c
    ${COMMON_SOURCE_DIR}/arp/arp.c
    )

INCLUDE_DIRECTORIES(
                    ${DEMO_SOURCE_DIR}
                    ${OBJDICT_DIR}/${OBJDICT}
                    ${COMMON_SOURCE_DIR}
                   )

ADD_DEFINITIONS(-DNMT_MAX_NODE_ID=254)
ADD_DEFINITIONS(-DCONFIG_INCLUDE_PDO)
ADD_DEFINITIONS(-DCONFIG_INCLUDE_SDO_ASND)

IF(CFG_IP_STACK)
    ADD_DEFINITIONS(-DCONFIG_INCLUDE_IP)
    ADD_DEFINITIONS(-DCONFIG_INCLUDE_SDO_UDP)
ENDIF(CFG_IP_STACK)

ADD_DEFINITIONS(-DCONFIG_INCLUDE_CFM)

# Redundant MN is not supported on embedded targets (yet)
#ADD_DEFINITIONS(-DCONFIG_INCLUDE_NMT_RMN)

#ADD_DEFINITIONS(-DCONFIG_APP_STORE_RESTORE)
#ADD_DEFINITIONS(-DCONFIG_DLL_PRES_CHAINING_CN)
#ADD_DEFINITIONS(-DCONFIG_INCLUDE_MASND)

################################################################################
# Setup the architecture specific definitions
IF (CMAKE_SYSTEM_NAME STREQUAL "Generic" AND CMAKE_SYSTEM_PROCESSOR STREQUAL "Microblazeise")
    include(microblazeise.cmake)
ELSEIF (CMAKE_SYSTEM_NAME STREQUAL "Generic" AND CMAKE_SYSTEM_PROCESSOR STREQUAL "alterac5arm")
    include(c5socarm.cmake)
ELSE ()
    MESSAGE(FATAL_ERROR "System ${CMAKE_SYSTEM_NAME} with processor ${CMAKE_SYSTEM_PROCESSOR} is not supported!")
ENDIF ()

SET(EXECUTABLE_NAME "${PROJECT_NAME}${ARCH_EXE_SUFFIX}")

################################################################################
# Setup compile definitions depending on configuration

IF(CFG_CDC_ON_SD)
    ADD_DEFINITIONS(-DCONFIG_CDC_ON_SD=TRUE)
ELSE()
    ADD_DEFINITIONS(-DCONFIG_CDC_ON_SD=FALSE)
ENDIF()

IF (CFG_KERNEL_STACK_DIRECTLINK)
    ADD_DEFINITIONS(-DCONFIG_KERNELSTACK_DIRECTLINK)
ENDIF (CFG_KERNEL_STACK_DIRECTLINK)

################################################################################
# Set the executable
ADD_EXECUTABLE(${EXECUTABLE_NAME} ${DEMO_SOURCES} ${DEMO_ARCH_SOURCES} ${CMAKE_CURRENT_BINARY_DIR}/mnobd.cdc)
SET_PROPERTY(TARGET ${EXECUTABLE_NAME}
             PROPERTY COMPILE_DEFINITIONS_DEBUG DEBUG;DEF_DEBUG_LVL=${CFG_DEBUG_LVL})

SET_PROPERTY(TARGET ${EXECUTABLE_NAME}
             PROPERTY LINK_FLAGS ${ARCH_LINKER_FLAGS})

ADD_CUSTOM_COMMAND(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/mnobd.cdc
                   COMMAND ${CMAKE_COMMAND} -E copy ${OPENCONFIG_PROJ_DIR}/Demo_3CN/output/mnobd.cdc ${CMAKE_CURRENT_BINARY_DIR}/mnobd.cdc
                   DEPENDS ${OPENCONFIG_PROJ_DIR}/Demo_3CN/output/mnobd.cdc
                   VERBATIM
                   )

################################################################################
# Libraries to link

OPLK_LINK_LIBRARIES(${EXECUTABLE_NAME})
TARGET_LINK_LIBRARIES(${EXECUTABLE_NAME} ${ARCH_LIBRARIES})

################################################################################
# Post Build targets
IF (CMAKE_SYSTEM_NAME STREQUAL "Generic" AND CMAKE_SYSTEM_PROCESSOR STREQUAL "Microblazeise")
   INCLUDE(microblazeapppostactionsise)
ELSEIF (CMAKE_SYSTEM_NAME STREQUAL "Generic" AND CMAKE_SYSTEM_PROCESSOR STREQUAL "alterac5arm")
   INCLUDE(c5socarmapppostactions)
ENDIF ()

################################################################################
# Add additional clean files
SET_DIRECTORY_PROPERTIES( PROPERTIES ADDITIONAL_MAKE_CLEAN_FILES "${ADD_CLEAN_FILES}" )

################################################################################
# Installation rules
INSTALL(TARGETS ${EXECUTABLE_NAME} RUNTIME DESTINATION ${CFG_DEMO_BOARD_NAME}/${CFG_DEMO_NAME})
INSTALL(FILES ${CMAKE_BINARY_DIR}/mnobd.cdc DESTINATION ${CFG_DEMO_BOARD_NAME}/${CFG_DEMO_NAME})
