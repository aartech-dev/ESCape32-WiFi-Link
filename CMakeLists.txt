cmake_minimum_required(VERSION 3.16)

# Product brand for this build — selects UI branding, color theme, font,
# built-in motor catalog, and default AP identity in main/CMakeLists.txt
# and main/root.html. Override with: idf.py build -DBRAND=nsr
set(BRAND "aart" CACHE STRING "Product brand: aart or nsr")
set_property(CACHE BRAND PROPERTY STRINGS aart nsr)
if(NOT BRAND STREQUAL "aart" AND NOT BRAND STREQUAL "nsr")
	message(FATAL_ERROR "Unknown BRAND '${BRAND}' — expected 'aart' or 'nsr'")
endif()

# Version scheme: <upstream-base>-aart.<aart-revision>
# Tracks neoxic/ESCape32-WiFi-Link upstream; increment aart.N for AART-only
# changes. Shared by both brands — only branding/config differs by BRAND.
set(PROJECT_VER "1.10-aart.1")
include($ENV{IDF_PATH}/tools/cmake/project.cmake)
if(BRAND STREQUAL "nsr")
	project(nsr-remora3-programmer)
else()
	project(aart-remora-programmer)
endif()
