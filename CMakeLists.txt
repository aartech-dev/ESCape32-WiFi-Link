cmake_minimum_required(VERSION 3.16)

# Product brand for this build — selects UI branding, color theme, font,
# built-in motor catalog, and default AP identity in main/CMakeLists.txt
# and main/root.html. Override with: idf.py build -DBRAND=nsr
set(BRAND "aart" CACHE STRING "Product brand: aart or nsr")
set_property(CACHE BRAND PROPERTY STRINGS aart nsr)
if(NOT BRAND STREQUAL "aart" AND NOT BRAND STREQUAL "nsr")
	message(FATAL_ERROR "Unknown BRAND '${BRAND}' — expected 'aart' or 'nsr'")
endif()

# Version scheme: <upstream-base>.<brand>.<brand-revision>
# Tracks neoxic/ESCape32-WiFi-Link upstream in <upstream-base>; increment
# <brand-revision> independently per brand for brand-relevant changes.
set(PROJECT_VER_BASE "1.10")
if(BRAND STREQUAL "nsr")
	set(PROJECT_VER "${PROJECT_VER_BASE}.nsr.1")
else()
	set(PROJECT_VER "${PROJECT_VER_BASE}.aart.1")
endif()
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
if(BRAND STREQUAL "nsr")
	project(nsr-programmer)
else()
	project(aart-remora-programmer)
endif()
