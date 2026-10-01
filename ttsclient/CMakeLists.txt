################################################################################
# If not stated otherwise in this file or this component's LICENSE file the
# following copyright and licenses apply:
#
# Copyright 2019 RDK Management
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
# http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
################################################################################

find_package(Thunder QUIET)

if (NOT Thunder_FOUND)
    find_package(WPEFramework REQUIRED)
    add_definitions(-DThunder=${NAMESPACE})
endif()

set(THUNDER_CONFIG_PATH "/etc/${NAMESPACE}/config.json" CACHE STRING "Path to config file")

pkg_check_modules(THUNDER_PLUGINS REQUIRED ${NAMESPACE}Plugins)
pkg_check_modules(THUNDER_SECURITYUTIL ${NAMESPACE}SecurityUtil)

# Build TTS Service Client library
set(TextToSpeechServiceClient_SOURCES
    TextToSpeechServiceCOMRPC.cpp
    TextToSpeechService.cpp
    Service.cpp
    ../common/logger.cpp
)

if(TTS_DEFAULT_BACKEND STREQUAL "firebolt")
    list(APPEND TextToSpeechServiceClient_SOURCES TextToSpeechServiceFirebolt.cpp)
endif()

add_library(TextToSpeechServiceClient SHARED ${TextToSpeechServiceClient_SOURCES})

target_compile_definitions(TextToSpeechServiceClient PRIVATE THUNDER_CONFIG_PATH="${THUNDER_CONFIG_PATH}")

# Build TTS Service Client library
set(TTSClient_SOURCES
    TTSClient.cpp
    TTSClientPrivateJsonRPC.cpp
    TTSClientPrivateCOMRPC.cpp
)

if(TTS_DEFAULT_BACKEND STREQUAL "firebolt")
    add_definitions(-DTTS_DEFAULT_BACKEND_FIREBOLT)
    list(APPEND TTSClient_SOURCES TTSClientPrivateFirebolt.cpp)
endif()

add_library(TTSClient SHARED ${TTSClient_SOURCES})

target_include_directories(TextToSpeechServiceClient PUBLIC ${THUNDER_PLUGINS_INCLUDE_DIRS})
target_include_directories(TTSClient PUBLIC ${GLIB_INCLUDE_DIRS})

if(NOT THUNDER_SECURITYUTIL_FOUND)
    add_definitions(-DSECURITY_TOKEN_ENABLED=0)
endif()

if(TTS_DEFAULT_BACKEND)
    add_definitions(-DTTS_DEFAULT_BACKEND="${TTS_DEFAULT_BACKEND}")
endif()

if(TTS_DEFAULT_BACKEND STREQUAL "firebolt")
    target_link_libraries(TextToSpeechServiceClient PUBLIC
        ${THUNDER_PLUGINS_LIBRARIES}
        ${THUNDER_SECURITYUTIL_LIBRARIES}
        -lpthread
        FireboltSDK
    )
else()
    target_link_libraries(TextToSpeechServiceClient PUBLIC
        ${THUNDER_PLUGINS_LIBRARIES}
        ${THUNDER_SECURITYUTIL_LIBRARIES}
        -lpthread
    )
endif()

target_link_libraries(TTSClient PUBLIC
    ${GLIB_LIBRARIES}
    ${LIBS}
    TextToSpeechServiceClient
)

target_compile_options(TextToSpeechServiceClient PRIVATE
    -Wno-error=unused-parameter)

install(TARGETS TTSClient TextToSpeechServiceClient LIBRARY DESTINATION lib)
install(FILES TTSClient.h ../common/TTSCommon.h TextToSpeechService.h Service.h DESTINATION include)
