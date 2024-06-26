cmake_minimum_required(VERSION 3.5)

project(qvtf)

option(QT4VTF "Build for Qt4" ${QT4VTF})
option(QT5VTF "Build for Qt5" ON) # Default to ON for now.

# Don't use system paths, let us install locally!
#set(KDE_INSTALL_USE_QT_SYS_PATHS FALSE)

add_subdirectory(external/vtflib)

# ---- dependencies -----------------------------------------------------------

if (QT4VTF)
	find_package(Qt4 REQUIRED)
	find_package(KDE4) # optional, only for extra .desktop file installation
else()
	set(CMAKE_AUTOMOC ON)
	set(CMAKE_INCLUDE_CURRENT_DIR ON)
	if (QT5VTF)
		find_package(Qt5Gui REQUIRED)
		set(QTDIR "qt5")
	else()
		find_package(Qt6Gui REQUIRED)
		set(QTDIR "qt6")
	endif()
	
	find_package(ECM 5.9.0  NO_MODULE)
	if (${ECM_FOUND})
		set(CMAKE_MODULE_PATH ${ECM_MODULE_PATH} ${ECM_KDE_MODULE_DIR})
		include(KDEInstallDirs)
		include(KDEFrameworkCompilerSettings)
		include(KDECMakeSettings)
	endif()
endif()

find_program(XDG_MIME_EXECUTABLE xdg-mime) # optional, for MIME type installation

# ---- compiler flags ---------------------------------------------------------
if(MSVC)
	# Force to always compile with W4
	if(CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
		string(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
	else()
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
	endif()
elseif(CMAKE_COMPILER_IS_GNUCC OR CMAKE_COMPILER_IS_GNUCXX)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Wextra -pedantic -Wno-write-strings -Wno-suggest-override -O3")
endif()

if(NOT(${CMAKE_SYSTEM_NAME} STREQUAL "Windows"))
	add_definitions(-D_FILE_OFFSET_BITS=64 -D_LARGEFILE_SOURCE)
endif()

# ---- build library ----------------------------------------------------------
if (QT4VTF)
	include(${QT_USE_FILE})
	add_definitions(${QT_DEFINITIONS})
	qt4_wrap_cpp(vtf_MOC ValveTextureFile.h)
	add_library(qvtf SHARED ValveTextureFile.cpp ValveTextureFile.h ${vtf_MOC})
	target_link_libraries(qvtf ${QT_LIBRARIES} vtflib_static)
else()
	add_library(qvtf SHARED ValveTextureFile.cpp ValveTextureFile.h ${vtf_MOC})
	if (QT5VTF)
		target_link_libraries(qvtf Qt5::Gui Qt5::Core vtflib_static)
	else()
		target_link_libraries(qvtf Qt6::Gui Qt6::Core vtflib_static)
	endif()
endif()

# ---- install target ---------------------------------------------------------
if (QT4VTF)
	install(TARGETS qvtf DESTINATION "${QT_IMAGEFORMATS_PLUGINS_DIR}")
else()
	include(GNUInstallDirs)
	install(TARGETS qvtf DESTINATION "${CMAKE_INSTALL_LIBDIR}/${QTDIR}/plugins/imageformats") # Does Qt5 cmake module provide this path as sole variable?
endif()

if (${ECM_FOUND})
	if (QT5VTF)
		install(FILES vtf.desktop DESTINATION "${KDE_INSTALL_KSERVICES5DIR}/qimageioplugins")
		install(FILES vtfthumbnail.desktop DESTINATION ${KDE_INSTALL_KSERVICES5DIR})
	else() # This probably needs changing, kf6 ECM is WIP, though.
		install(FILES vtf.desktop DESTINATION "${KDE_INSTALL_PLUGINDIR}/qimageioplugins")
		install(FILES vtfthumbnail.desktop DESTINATION ${KDE_INSTALL_APPDIR})
	endif()
elseif(${KDE4_FOUND})
	install(FILES vtf.desktop DESTINATION "${SERVICES_INSTALL_DIR}/qimageioplugins")
	install(FILES vtfthumbnail.desktop DESTINATION ${SERVICES_INSTALL_DIR})
else()
# ---- uninstall target (KDE creates it for us) -------------------------------
	configure_file(
		"${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
		"${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
		IMMEDIATE @ONLY)

	add_custom_target(uninstall
		COMMAND ${CMAKE_COMMAND} -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")
endif()

if(NOT(${XDG_MIME_EXECUTABLE} STREQUAL "XDG_MIME_EXECUTABLE-NOTFOUND"))
	install(CODE "execute_process(COMMAND ${XDG_MIME_EXECUTABLE} install --novendor \"${PROJECT_SOURCE_DIR}/vtf.xml\")")
endif()
