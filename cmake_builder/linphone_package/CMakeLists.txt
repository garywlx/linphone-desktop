############################################################################
# CMakeLists.txt
# Copyright (C) 2017  Belledonne Communications, Grenoble France
#
############################################################################
#
# This program is free software; you can redistribute it and/or
# modify it under the terms of the GNU General Public License
# as published by the Free Software Foundation; either version 2
# of the License, or (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA  02110-1301, USA.
#
############################################################################

cmake_minimum_required(VERSION 3.0)
project(LINPHONE_PACKAGE) # Dummy project

find_package(Linphone REQUIRED)
find_package(Git)

set(LINPHONE_QML_DIR "${LINPHONE_DESKTOP_DIR}/ui")

if(WIN32)
	set(WINDEPLOYQT_PROGRAM windeployqt)
endif()

if(GIT_EXECUTABLE)
	execute_process(
		COMMAND ${GIT_EXECUTABLE} describe --always
		OUTPUT_VARIABLE LINPHONE_GIT_REVISION
		OUTPUT_STRIP_TRAILING_WHITESPACE
		WORKING_DIRECTORY ${LINPHONE_SOURCE_DIR}
	)
else()
	set(LINPHONE_GIT_REVISION "0.0.0")
endif()
string(REGEX REPLACE "([0-9.]+)-?.*" "\\1" LINPHONE_VERSION "${LINPHONE_GIT_REVISION}")
string(REPLACE "." ";" SPLITTED_LINPHONE_VERSION ${LINPHONE_VERSION})
list(GET SPLITTED_LINPHONE_VERSION 0 LINPHONE_MAJOR_VERSION)
list(GET SPLITTED_LINPHONE_VERSION 1 LINPHONE_MINOR_VERSION)
list(GET SPLITTED_LINPHONE_VERSION 2 LINPHONE_MICRO_VERSION)
set(PACKAGE_VERSION "${LINPHONE_VERSION}")

# Preparing the Linphone SDK bundle
set(LINPHONE_SDK_TMP "${CMAKE_CURRENT_BINARY_DIR}/linphone-sdk-tmp")
file(REMOVE_RECURSE "${LINPHONE_SDK_TMP}")
file(COPY "${LINPHONE_OUTPUT_DIR}" DESTINATION "${CMAKE_CURRENT_BINARY_DIR}")
file(RENAME "${CMAKE_CURRENT_BINARY_DIR}/desktop" "${LINPHONE_SDK_TMP}")
if(APPLE)
	set(RPATH_CHANGE_CMDS
		-change /opt/local/lib/libintl.8.dylib @rpath/libintl.8.dylib
		-change /opt/local/lib/libiconv.2.dylib @rpath/libiconv.2.dylib
	)
	file(COPY
		/opt/local/lib/libintl.8.dylib
		/opt/local/lib/libintl.dylib
		/opt/local/lib/libiconv.dylib
		/opt/local/lib/libiconv.2.dylib
		DESTINATION ${LINPHONE_SDK_TMP}/lib
	)
	execute_process(COMMAND install_name_tool
		-id @rpath/libintl.8.dylib
		-change /opt/local/lib/libiconv.2.dylib @rpath/libiconv.2.dylib
		${LINPHONE_SDK_TMP}/lib/libintl.8.dylib
	)
	execute_process(COMMAND install_name_tool
		-id @rpath/libiconv.2.dylib
		${LINPHONE_SDK_TMP}/lib/libiconv.2.dylib
	)
	execute_process(COMMAND install_name_tool
		-change /opt/local/lib/libintl.8.dylib @rpath/libintl.8.dylib
		-change /opt/local/lib/libiconv.2.dylib @rpath/libiconv.2.dylib
		${LINPHONE_SDK_TMP}/lib/liblinphone.dylib
	)
endif()

# Generates a zip archive containing the development files of liblinphone
macro(add_list_file NAME)
	configure_file("${CMAKE_CURRENT_SOURCE_DIR}/sdk_content/${NAME}.list.in" "${LINPHONE_SDK_TMP}/${NAME}.list" @ONLY)
	list(APPEND LIST_FILES "${LINPHONE_SDK_TMP}/${NAME}.list")
endmacro()
if(WIN32)
	set(LIBDIR "bin")
	set(LIBPREFIX "")
	set(LIBEXT "dll")
	set(PLUGINEXT "dll")
elseif(APPLE)
	set(LIBDIR "lib")
	set(LIBPREFIX "lib")
	set(LIBEXT "dylib")
	set(PLUGINEXT "so")
endif()
set(LIST_FILES )
add_list_file(bctoolbox)
add_list_file(antlr3c)
add_list_file(bellesip)
add_list_file(ortp)
add_list_file(ms2)
add_list_file(ms2plugins)
add_list_file(linphone)
if(WIN32)
	add_list_file(sqlite3)
	add_list_file(xml2)
	add_list_file(zlib)
endif()
if(ENABLE_UNIT_TESTS)
	add_list_file(bcunit)
endif()
if(ENABLE_VCARD)
	add_list_file(vcard)
endif()
if(ENABLE_BV16)
	add_list_file(bv16)
endif()
if(ENABLE_GSM)
	add_list_file(gsm)
endif()
if(ENABLE_OPUS)
	add_list_file(opus)
endif()
if(ENABLE_SPEEX)
	add_list_file(speex)
endif()
if(ENABLE_MBEDTLS)
	add_list_file(mbedtls)
endif()
if(ENABLE_POLARSSL)
	add_list_file(polarssl)
endif()
if(ENABLE_SRTP)
	add_list_file(srtp)
endif()
if(ENABLE_ZRTP)
	add_list_file(bzrtp)
endif()
if(ENABLE_FFMPEG)
	add_list_file(ffmpeg)
endif()
if(ENABLE_OPENH264)
	configure_file("${CMAKE_CURRENT_SOURCE_DIR}/README.openh264.txt.in" "${LINPHONE_SDK_TMP}/README.openh264.txt" @ONLY)
	add_list_file(openh264)
endif()
file(WRITE "${LINPHONE_SDK_TMP}/linphone-sdk.list" "")
foreach(_file ${LIST_FILES})
	file(READ "${_file}" _content)
	file(APPEND "${LINPHONE_SDK_TMP}/linphone-sdk.list" "${_content}\n")
endforeach()
if(WIN32)
	find_program(7Z_PROGRAM 7z.exe)
	if(7Z_PROGRAM)
		execute_process(
			COMMAND ${7Z_PROGRAM} a -tzip ${CMAKE_BINARY_DIR}/linphone-sdk-${LINPHONE_GIT_REVISION}-win32.zip @linphone-sdk.list
			WORKING_DIRECTORY ${LINPHONE_SDK_TMP}
		)
	else()
		message(WARNING "7z has not been found, cannot generate the SDK!")
	endif()
elseif(APPLE)
	execute_process(
		COMMAND rm -rf "${CMAKE_BINARY_DIR}/linphone-sdk-${LINPHONE_GIT_REVISION}-mac.zip"
		COMMAND zip -ry "${CMAKE_BINARY_DIR}/linphone-sdk-${LINPHONE_GIT_REVISION}-mac.zip" . -i "@linphone-sdk.list"
		WORKING_DIRECTORY ${LINPHONE_SDK_TMP}
	)
endif()

if(WIN32)
	execute_process(
		COMMAND ${WINDEPLOYQT_PROGRAM} "${LINPHONE_OUTPUT_DIR}/bin/linphone.exe" "--qmldir" "${LINPHONE_QML_DIR}"
	)

	file(GLOB SHARE_CONTENT RELATIVE "${LINPHONE_OUTPUT_DIR}" "${LINPHONE_OUTPUT_DIR}/share/*")
	list(REMOVE_ITEM SHARE_CONTENT "share/images" "share/linphone" "share/Linphone" "share/sounds")
	foreach(item IN LISTS SHARE_CONTENT)
		list(APPEND SHARE_CONTENT_EXCLUDE PATTERN "${item}" EXCLUDE)
	endforeach()

	install(DIRECTORY "${LINPHONE_OUTPUT_DIR}/"
		DESTINATION "."
		COMPONENT "linphone_base"
		PATTERN "bin/belcard*.exe" EXCLUDE
		PATTERN "bin/*_tester.exe" EXCLUDE
		PATTERN "bin/openh264.dll" EXCLUDE
		PATTERN "include" EXCLUDE
		PATTERN "lib/mediastreamer" EXCLUDE
		PATTERN "lib/pkgconfig" EXCLUDE
		PATTERN "lib/*.a" EXCLUDE
		PATTERN "lib/*.def" EXCLUDE
		PATTERN "lib/*.exp" EXCLUDE
		PATTERN "lib/*.la" EXCLUDE
		PATTERN "lib/*.lib" EXCLUDE
		PATTERN "lib/Win32" EXCLUDE
		PATTERN "linphone.lnk" EXCLUDE
		PATTERN "share/Linphone/cmake" EXCLUDE
		${SHARE_CONTENT_EXCLUDE}
	)


	if(ENABLE_OPENH264)
		# Include bunzip2.exe and bzip2.dll to be able to extract the openh264 file downloaded from Cisco
		install(DIRECTORY "${TOOLS_DIR}/"
			DESTINATION "bin"
			COMPONENT "msopenh264"
			FILES_MATCHING PATTERN "b*zip2.*"
		)
		install(DIRECTORY "${LINPHONE_OUTPUT_DIR}/lib/mediastreamer/plugins/"
			DESTINATION "lib/mediastreamer/plugins"
			COMPONENT "msopenh264"
			FILES_MATCHING PATTERN "libmsopenh264.*"
		)
	endif()

elseif(APPLE)
	find_program(MAC_BUNDLER gtk-mac-bundler)
	if(NOT MAC_BUNDLER)
		message(FATAL_ERROR "Could not find gtk-mac-bundler executable!")
	endif()
	find_program(PATCH_PROGRAM patch)
	if(NOT PATCH_PROGRAM)
		message(FATAL_ERROR "Could not find patch executable!")
	endif()
	configure_file("${LINPHONE_SOURCE_DIR}/build/macos/Info-linphone.plist.in" "mac-bundler/Info-linphone.plist" @ONLY)
	configure_file("${LINPHONE_SOURCE_DIR}/build/macos/environment.sh" "mac-bundler/environment.sh" COPYONLY)
	configure_file("${LINPHONE_SOURCE_DIR}/gtk/gtkrc.mac" "mac-bundler/gtkrc" COPYONLY)
	configure_file("${LINPHONE_SOURCE_DIR}/pixmaps/linphone.icns" "mac-bundler/linphone.icns" COPYONLY)
	configure_file("mac-bundler/linphone.bundle" "mac-bundler/linphone.bundle" COPYONLY)	# This file has to be different than the one in the linphone source code because of different file access paths
	configure_file("mac-bundler/pkg-scripts/postinstall.in" "mac-bundler/pkg-scripts/postinstall" @ONLY)
	configure_file("mac-bundler/call-bundler.cmake.in" "mac-bundler/call-bundler.cmake")
	install(SCRIPT "${CMAKE_BINARY_DIR}/mac-bundler/call-bundler.cmake")
endif()

if(MSVC)
	install(DIRECTORY "${LINPHONE_OUTPUT_DIR}/lib/mediastreamer/plugins/"
		DESTINATION "lib/mediastreamer/plugins"
		FILES_MATCHING PATTERN "libmswasapi.dll"
	)
	string(REGEX REPLACE "Visual Studio ([0-9]+).*" "\\1" MSVC_VERSION "${CMAKE_GENERATOR}")
	find_file(MSVCP_LIB "msvcp${MSVC_VERSION}0.dll" PATHS "C:/Windows/System32")
	find_file(UCRTBASE_LIB "ucrtbase.dll" PATHS "C:/Windows/System32")
	# Starting with Visual Studio 2015 (MSVC_VERSION==14) the msvcr dll has been renamed to vcruntime
	find_file(VCRUNTIME_LIB "vcruntime${MSVC_VERSION}0.dll" PATHS "C:/Windows/System32")
	if(NOT VCRUNTIME_LIB)
		find_file(VCRUNTIME_LIB "msvcr${MSVC_VERSION}0.dll" PATHS "C:/Windows/System32")
	endif()
	install(FILES ${MSVCP_LIB} ${UCRTBASE_LIB} ${VCRUNTIME_LIB} DESTINATION "bin")
	if(CMAKE_BUILD_TYPE STREQUAL "Debug")
		find_file(MSVCPD_LIB "msvcp${MSVC_VERSION}0d.dll" PATHS "C:/Windows/System32")
		find_file(UCRTBASED_LIB "ucrtbased.dll" PATHS "C:/Windows/System32")
		find_file(VCRUNTIMED_LIB "vcruntime${MSVC_VERSION}0d.dll" PATHS "C:/Windows/System32")
		if(NOT VCRUNTIMED_LIB)
			find_file(VCRUNTIMED_LIB "msvcr${MSVC_VERSION}0d.dll" PATHS "C:/Windows/System32")
		endif()
		install(FILES ${MSVCPD_LIB} ${UCRTBASED_LIB} ${VCRUNTIMED_LIB} DESTINATION "bin")
	endif()
endif()

set(CPACK_PACKAGE_NAME "Linphone")
set(CPACK_PACKAGE_VENDOR "Belledonne communications")
set(CPACK_PACKAGE_VERSION_MAJOR ${LINPHONE_MAJOR_VERSION})
set(CPACK_PACKAGE_VERSION_MINOR ${LINPHONE_MINOR_VERSION})
set(CPACK_PACKAGE_VERSION_PATCH ${LINPHONE_MICRO_VERSION})
set(CPACK_PACKAGE_EXECUTABLES "linphone;Linphone")
set(CPACK_PACKAGE_INSTALL_DIRECTORY "Linphone")
set(CPACK_RESOURCE_FILE_LICENSE "${LINPHONE_SOURCE_DIR}/COPYING")

if(WIN32)
	set(CPACK_COMPONENT_LINPHONE_BASE_DISPLAY_NAME "Linphone")
	set(CPACK_COMPONENT_LINPHONE_BASE_DESCRIPTION "Minimal installation of Linphone")
	set(CPACK_COMPONENT_LINPHONE_BASE_REQUIRED True)
	if(ENABLE_OPENH264)
		set(CPACK_COMPONENT_MSOPENH264_DISPLAY_NAME "Cisco's OpenH264 codec")
		set(CPACK_COMPONENT_MSOPENH264_DESCRIPTION "Download OpenH264 Video Codec provided by Cisco Systems, Inc.")
		set(CPACK_COMPONENT_MSOPENH264_DISABLED True)
	endif()
endif()

if(WIN32)
	# Use template from the current source directory to force uninstallation and ensure that linphone is not running before installation
	set(CPACK_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}")
	set(CPACK_GENERATOR "NSIS")
	set(CPACK_PACKAGE_ICON "${CMAKE_CURRENT_SOURCE_DIR}\\\\linphone-banner.bmp")
	set(CPACK_PACKAGE_FILE_NAME "${CPACK_PACKAGE_NAME}-${LINPHONE_GIT_REVISION}-win32")
	set(CPACK_NSIS_MUI_ICON "${LINPHONE_DESKTOP_DIR}/linphone.ico")
	set(CPACK_NSIS_MUI_UNIICON "${LINPHONE_DESKTOP_DIR}/linphone.ico")
	set(CPACK_NSIS_DISPLAY_NAME "Linphone")
	set(CPACK_NSIS_PACKAGE_NAME "Linphone ${LINPHONE_MAJOR_VERSION}.${LINPHONE_MINOR_VERSION}.${LINPHONE_MICRO_VERSION}")
	set(CPACK_NSIS_URL_INFO_ABOUT "http://www.linphone.org/")

	file(TO_NATIVE_PATH "${CMAKE_CURRENT_SOURCE_DIR}" DOS_STYLE_SOURCE_DIR)
	string(REPLACE "\\" "\\\\" ESCAPED_DOS_STYLE_SOURCE_DIR "${DOS_STYLE_SOURCE_DIR}")
	file(TO_NATIVE_PATH "${CMAKE_CURRENT_BINARY_DIR}" DOS_STYLE_BINARY_DIR)
	string(REPLACE "\\" "\\\\" ESCAPED_DOS_STYLE_BINARY_DIR "${DOS_STYLE_BINARY_DIR}")

	configure_file("${CMAKE_CURRENT_SOURCE_DIR}/uninstall.nsi" "${CMAKE_CURRENT_BINARY_DIR}/uninstall.nsi" COPYONLY)
	set(CPACK_NSIS_EXTRA_UNINSTALL_COMMANDS "!include \\\"${ESCAPED_DOS_STYLE_BINARY_DIR}\\\\uninstall.nsi\\\"")
	if(ENABLE_OPENH264)
		# Add NSI instructions to download the openh264 codec from Cisco's servers whether msopenh264 has been installed
		configure_file("${CMAKE_CURRENT_SOURCE_DIR}/openh264-install.nsi.in" "${CMAKE_CURRENT_BINARY_DIR}/openh264-install.nsi" @ONLY)
		set(CPACK_NSIS_EXTRA_INSTALL_COMMANDS "!include \\\"${ESCAPED_DOS_STYLE_BINARY_DIR}\\\\openh264-install.nsi\\\"")

		# Add NSI instructions to delete the openh264 codecs while uninstalling Linphone
		set(CPACK_NSIS_EXTRA_UNINSTALL_COMMANDS "${CPACK_NSIS_EXTRA_UNINSTALL_COMMANDS}\n!include \\\"${ESCAPED_DOS_STYLE_SOURCE_DIR}\\\\openh264-uninstall.nsi\\\"")
	endif()

	# Sign the installer
	set(TIMESTAMP_URL "http://timestamp.verisign.com/scripts/timestamp.dll")
	set(PFX_FILE "${CMAKE_CURRENT_SOURCE_DIR}/sign/linphone.pfx")
	set(PASSPHRASE_FILE "${CMAKE_CURRENT_SOURCE_DIR}/sign/passphrase.txt")
	get_filename_component(WINSDK_DIR "[HKEY_LOCAL_MACHINE\\SOFTWARE\\Microsoft\\Microsoft SDKs\\Windows;CurrentInstallFolder]" REALPATH CACHE)
	find_package(PythonInterp)
	find_program(SIGNTOOL signtool PATHS ${WINSDK_DIR}/bin)
	set(SIGNTOOL_COMMAND "${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/signtool.py")
	set(PERFORM_SIGNING 0)
	if(EXISTS ${PFX_FILE})
		if(SIGNTOOL)
			set(SIGNTOOL_COMMAND "${SIGNTOOL_COMMAND} signtool sign /f ${PFX_FILE}")
			set(SIGNTOOL_COMMAND "${SIGNTOOL_COMMAND} /p ${PASSPHRASE_FILE}")
			set(SIGNTOOL_COMMAND "${SIGNTOOL_COMMAND} /t ${TIMESTAMP_URL}")
			message("Found signtool and certificate ${PFX_FILE}")
			set(PERFORM_SIGNING 1)
		else()
			message(STATUS "Could not find signtool! Code signing disabled (${SIGNTOOL})")
		endif()
	else()
		message(STATUS "No signtool certificate found; assuming development machine (${PFX_FILE})")
	endif()
	configure_file("${CMAKE_CURRENT_SOURCE_DIR}/WindowsPackaging.cmake.in" "${CMAKE_CURRENT_BINARY_DIR}/WindowsPackaging.cmake" @ONLY)
	install(SCRIPT "${CMAKE_CURRENT_BINARY_DIR}/WindowsPackaging.cmake")
endif()
include(CPack)