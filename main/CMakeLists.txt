idf_component_register(SRCS main.c PRIV_REQUIRES espressif__mdns)
set(lang_map en:EN fr:FR de:DE it:IT es:ES pt:PT nl:NL sv:SV da:DA uk:УК lv:LV fi:FI et:ET cs:CS pl:PL lt:LT zh:中)
string(REGEX REPLACE "([a-z][a-z]):([^;]+)" "XX(\\1)" LANG_LIST "${lang_map}")
string(REGEX REPLACE "([a-z][a-z]):([^;]+)" "<option value=\"\\1\">\\2</option>" LANG_OPTS "${lang_map}")
list(JOIN LANG_LIST " " LANG_LIST)
list(JOIN LANG_OPTS "" LANG_OPTS)

# Inline only English into @LANG_DATA@ in root.html, for an instant first
# paint with no network round-trip. Every other language (including a
# non-English browser default) is fetched on demand from GET /?<lang> —
# see setlang() in root.html. Embedding all of them here stopped scaling
# once the language count grew: at 17 languages it was adding ~37KB to the
# compressed image for translations most sessions never touch.
file(READ "${CMAKE_CURRENT_SOURCE_DIR}/root_en.json" _en_json_raw)
string(STRIP "${_en_json_raw}" _en_json_raw)
set(LANG_DATA "{\"en\":${_en_json_raw}}")

configure_file(build_defs.h.in build_defs.h @ONLY)
target_include_directories(${COMPONENT_TARGET} PRIVATE ${CMAKE_CURRENT_BINARY_DIR})
file(GLOB files root*.*)
foreach(file ${files})
	get_filename_component(name ${file} NAME)
	configure_file(${name} ${name} @ONLY)
	add_custom_command(
		OUTPUT ${name}.gz
		COMMAND gzip -f ${name}
		DEPENDS ${name}
		WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
	)
	target_add_binary_data(${COMPONENT_TARGET} ${CMAKE_CURRENT_BINARY_DIR}/${name}.gz BINARY DEPENDS ${name}.gz)
endforeach()
