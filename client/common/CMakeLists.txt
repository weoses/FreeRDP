# FreeRDP: A Remote Desktop Protocol Implementation
# FreeRDP Client Common
#
# Copyright 2012 Marc-Andre Moreau <marcandre.moreau@gmail.com>
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

set(MODULE_NAME "freerdp-client")
set(MODULE_PREFIX "FREERDP_CLIENT")

#  Policy CMP0022: INTERFACE_LINK_LIBRARIES defines the link
#  interface.  Run "cmake --help-policy CMP0022" for policy details.  Use the
#  cmake_policy command to set the policy and suppress this warning.
if(POLICY CMP0022)
  cmake_policy(SET CMP0022 NEW)
endif()

set(SRCS
	client.c
	client_rails.c
	cmdline.c
	cmdline.h
	file.c
	client_cliprdr_file.c
	geometry.c
	smartcard_cli.c)

foreach(FREERDP_CHANNELS_CLIENT_SRC ${FREERDP_CHANNELS_CLIENT_SRCS})
	get_filename_component(NINC ${FREERDP_CHANNELS_CLIENT_SRC} PATH)
	include_directories(${NINC})
	list(APPEND SRCS "${FREERDP_CHANNELS_CLIENT_SRC}")
endforeach()

if (NOT APPLE AND NOT WIN32 AND NOT ANDROID)
	set(OPT_FUSE_DEFAULT ON)
else()
	set(OPT_FUSE_DEFAULT OFF)
endif()

option(WITH_FUSE "Build clipboard with FUSE file copy support" ${OPT_FUSE_DEFAULT})
if(WITH_FUSE)
	include(FindPkgConfig)
	find_package(PkgConfig REQUIRED)

	pkg_check_modules(FUSE3 fuse3)
	if (FUSE3_FOUND)
		message("[FUSE3] using ${FUSE3_LIBRARIES}")

		include_directories(${FUSE3_INCLUDE_DIRS})
		add_definitions(-DWITH_FUSE3)
		list(APPEND LIBS ${FUSE3_LIBRARIES})
	else()
		pkg_check_modules(FUSE2 REQUIRED fuse)

		message("[FUSE2] using ${FUSE2_LIBRARIES}")
		add_definitions(-DWITH_FUSE2)
		include_directories(${FUSE2_INCLUDE_DIRS})
		list(APPEND LIBS ${FUSE2_LIBRARIES})
	endif()

	add_definitions(-D_FILE_OFFSET_BITS=64)
endif()

# On windows create dll version information.
# Vendor, product and year are already set in top level CMakeLists.txt
if (WIN32 AND BUILD_SHARED_LIBS)
  set (RC_VERSION_MAJOR ${FREERDP_VERSION_MAJOR})
  set (RC_VERSION_MINOR ${FREERDP_VERSION_MINOR})
  set (RC_VERSION_BUILD ${FREERDP_VERSION_REVISION})
  set (RC_VERSION_FILE "${CMAKE_SHARED_LIBRARY_PREFIX}${MODULE_NAME}${FREERDP_API_VERSION}${CMAKE_SHARED_LIBRARY_SUFFIX}" )

  configure_file(
	${PROJECT_SOURCE_DIR}/cmake/WindowsDLLVersion.rc.in
	${CMAKE_CURRENT_BINARY_DIR}/version.rc
	@ONLY)

list (APPEND SRCS ${CMAKE_CURRENT_BINARY_DIR}/version.rc)
endif()

include_directories(${OPENSSL_INCLUDE_DIR})

add_library(${MODULE_NAME} ${SRCS})

set_target_properties(${MODULE_NAME} PROPERTIES OUTPUT_NAME ${MODULE_NAME}${FREERDP_API_VERSION})
if (WITH_LIBRARY_VERSIONING)
  set_target_properties(${MODULE_NAME} PROPERTIES VERSION ${FREERDP_VERSION} SOVERSION ${FREERDP_API_VERSION})
endif()

list(APPEND LIBS freerdp winpr)

target_include_directories(${MODULE_NAME} INTERFACE $<INSTALL_INTERFACE:include>)
target_link_libraries(${MODULE_NAME} PRIVATE ${FREERDP_CHANNELS_CLIENT_LIBS})
target_link_libraries(${MODULE_NAME} PUBLIC ${LIBS})

install(TARGETS ${MODULE_NAME} COMPONENT libraries EXPORT FreeRDP-ClientTargets
	ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
	LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
	RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

if (WITH_DEBUG_SYMBOLS AND MSVC AND BUILD_SHARED_LIBS)
	get_target_property(OUTPUT_FILENAME ${MODULE_NAME} OUTPUT_NAME)
	install(FILES ${CMAKE_PDB_BINARY_DIR}/${OUTPUT_FILENAME}.pdb DESTINATION ${CMAKE_INSTALL_LIBDIR} COMPONENT symbols)
endif()

set_property(TARGET ${MODULE_NAME} PROPERTY FOLDER "Client/Common")

if(BUILD_TESTING)
	add_subdirectory(test)
endif()
