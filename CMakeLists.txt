cmake_minimum_required(VERSION 3.16...3.21)

project(linux-kmsgrab VERSION 0.1.0)
add_library(${CMAKE_PROJECT_NAME} MODULE)

# Replace `Your Name Here` with the name (yours or your organization's) you want to see as the
# author of the plugin (in the plugin's metadata itself and in the installers)
set(PLUGIN_AUTHOR "Ivan Avdeev")

# Replace `me@contoso.com` with the maintainer email address you want to put in Linux packages
set(LINUX_MAINTAINER_EMAIL "me@w23.ru")

if(NOT UNIX OR APPLE)
	message(FATAL_ERROR "This plugin is Linux-only")
endif()

# Add your custom source files here - header files are optional and only required for visibility
# e.g. in Xcode or Visual Studio
target_sources(${CMAKE_PROJECT_NAME}
	PRIVATE
		src/dmabuf.c
		src/drm-helpers.c
		src/xcursor-xcb.c)

# Import libobs as main plugin dependency
find_package(libobs REQUIRED)
include(cmake/ObsPluginHelpers.cmake)

# Uncomment these lines if you want to use the OBS Frontend API in your plugin
find_package(obs-frontend-api REQUIRED)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE OBS::obs-frontend-api)

# Uncomment those lines if you want to use Qt in your plugin
#[[
find_qt(COMPONENTS Widgets Core)
target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE Qt::Core Qt::Widgets)
set_target_properties(
  ${CMAKE_PROJECT_NAME}
  PROPERTIES AUTOMOC ON
             AUTOUIC ON
             AUTORCC ON)
#]]

# LOL TODO for this we need cmake >= 3.19 w/ cmake policy CMP0109 set to NEW
#find_program(POLKIT NAMES pkexec)
option(ENABLE_POLKIT "Use pkexec for elevated drmsend privileges" ON)

find_package(PkgConfig)
pkg_check_modules(DRM libdrm)

if(NOT DRM_FOUND)
	message(FATAL_ERROR "libdrm(-dev) not found")
	return()
endif()

set(OBS_PLUGIN_DESTINATION "${CMAKE_INSTALL_LIBDIR}/obs-plugins")

if (ENABLE_POLKIT)
	message(STATUS "Using Polkit/pkexec for elevating drmsend privileges")
	target_compile_definitions(${CMAKE_PROJECT_NAME} PRIVATE USE_PKEXEC)
else()
	message(WARNING "Polkit support is disabled. You'll need to manually run `sudo setcap cap_sys_admin+ep \"${OBS_PLUGIN_DESTINATION}/obs-plugins/linux-kmsgrab-send\"` to allow it to grab framebuffers")
endif()

target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE xcb xcb-xfixes)

configure_file(src/plugin-macros.h.in ${CMAKE_SOURCE_DIR}/src/plugin-macros.generated.h)

target_sources(${CMAKE_PROJECT_NAME} PRIVATE src/plugin-macros.generated.h)

# /!\ TAKE NOTE: No need to edit things past this point /!\

# --- Platform-independent build settings ---

target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE ${CMAKE_SOURCE_DIR}/src ${DRM_INCLUDE_DIRS})

target_link_libraries(${CMAKE_PROJECT_NAME} PRIVATE OBS::libobs)

add_executable(linux-kmsgrab-send src/drmsend.c)
target_include_directories(linux-kmsgrab-send PRIVATE ${DRM_INCLUDE_DIRS})
target_link_libraries(linux-kmsgrab-send PRIVATE ${DRM_LIBRARIES})

install(TARGETS ${CMAKE_PROJECT_NAME} linux-kmsgrab-send
	LIBRARY DESTINATION "${OBS_PLUGIN_DESTINATION}"
	RUNTIME DESTINATION "${OBS_PLUGIN_DESTINATION}")

# --- End of section ---

# --- Windows-specific build settings and tasks ---
if(OS_WINDOWS)
  configure_file(cmake/bundle/windows/installer-Windows.iss.in
                 ${CMAKE_BINARY_DIR}/installer-Windows.generated.iss)

  configure_file(cmake/bundle/windows/resource.rc.in ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.rc)
  target_sources(${CMAKE_PROJECT_NAME} PRIVATE ${CMAKE_BINARY_DIR}/${CMAKE_PROJECT_NAME}.rc)

  if(MSVC)
    target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE /W4)
  endif()
  # --- End of section ---

  # -- macOS specific build settings and tasks --
elseif(OS_MACOS)
  configure_file(cmake/bundle/macos/installer-macos.pkgproj.in
                 ${CMAKE_BINARY_DIR}/installer-macos.generated.pkgproj)

  set(MACOSX_PLUGIN_GUI_IDENTIFIER "${MACOS_BUNDLEID}")
  set(MACOSX_PLUGIN_BUNDLE_VERSION "${CMAKE_PROJECT_VERSION}")
  set(MACOSX_PLUGIN_SHORT_VERSION_STRING "1")

  target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE -Wall)
  # --- End of section ---

  # --- Linux-specific build settings and tasks ---
else()
  target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE -Wall)
endif()
# --- End of section ---

setup_plugin_target(${CMAKE_PROJECT_NAME})

# NO
target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE -Wno-error)
