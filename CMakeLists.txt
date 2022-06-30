cmake_minimum_required(VERSION 3.22)
set(CMAKE_C_FLAGS_RELEASE "-Os -DNDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE "-Os -DNDEBUG")
set(CMAKE_C_FLAGS_DEBUG "-g -O0 -DDEBUG")
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -DDEBUG")
set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 20)
set(ignoreMe "${NO_PKGCFG}")
project(offsetfinder64)
if("${CMAKE_HOST_SYSTEM_NAME}" MATCHES "Darwin")
	if(NOT DEFINED NO_XCODE AND NOT DEFINED ENV{NO_XCODE})
	execute_process(COMMAND xcrun --find clang WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" OUTPUT_VARIABLE CC OUTPUT_STRIP_TRAILING_WHITESPACE)
	execute_process(COMMAND xcrun --find clang++ WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" OUTPUT_VARIABLE CXX OUTPUT_STRIP_TRAILING_WHITESPACE)
	execute_process(COMMAND xcrun --find ar WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" OUTPUT_VARIABLE AR OUTPUT_STRIP_TRAILING_WHITESPACE)
	execute_process(COMMAND xcrun --find ld WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" OUTPUT_VARIABLE LD OUTPUT_STRIP_TRAILING_WHITESPACE)
	execute_process(COMMAND xcrun --show-sdk-path WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" OUTPUT_VARIABLE SYSROOT OUTPUT_STRIP_TRAILING_WHITESPACE)
	set(CMAKE_C_COMPILER "${CC}")
	set(CMAKE_CXX_COMPILER "${CXX}")
	endif()
	if(NOT DEFINED ARCH)
	set(ARCH "$ENV{ARCH}")
	if(ARCH STREQUAL "")
		set(ARCH "${CMAKE_SYSTEM_PROCESSOR}")
	endif()
	endif()
	if(NOT DEFINED MINVER OR NOT DEFINED "$ENV{MINVER}" AND DEFINED NO_PKGCFG OR "$ENV{NO_PKGCFG}" MATCHES "1")
		if("${ARCH}" STREQUAL "x86_64" OR "$ENV{ARCH}" STREQUAL "x86_64")
			set(MINVER -mmacosx-version-min=10.12)
			set(CMAKE_OSX_DEPLOYMENT_TARGET 10.12)
		else()
			set(MINVER -mmacosx-version-min=11.0)
			set(CMAKE_OSX_DEPLOYMENT_TARGET 11.0)
		endif()
	endif()
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -arch ${ARCH} -isysroot ${SYSROOT} ${MINVER} -fembed-bitcode=off -flto=thin -pthread -mrelax-all -std=gnu++20")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -arch ${ARCH} -isysroot ${SYSROOT} ${MINVER} -fembed-bitcode=off -flto=thin -pthread -mrelax-all -std=gnu17")
elseif("${CMAKE_HOST_SYSTEM_NAME}" MATCHES "Linux")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wl,--allow-multiple-definition -pthread")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wl,--allow-multiple-definition -pthread")
endif()
add_library(offsetfinder64
        src/patchfinder64.cpp
		src/patch.cpp
		src/machopatchfinder64.cpp
		src/kernelpatchfinder64.cpp
		src/kernelpatchfinder64iOS13.cpp
		src/ibootpatchfinder64.cpp
		src/ibootpatchfinder64_base.cpp
		src/ibootpatchfinder64_iOS14.cpp)
target_include_directories(offsetfinder64 PRIVATE
        "${CMAKE_SOURCE_DIR}/dep_root/include"
        src
		include/liboffsetfinder64
		include)
target_link_directories(offsetfinder64 PRIVATE
        "${CMAKE_SOURCE_DIR}/dep_root/lib")
if(NOT DEFINED NO_PKGCFG AND NOT "$ENV{NO_PKGCFG}" MATCHES "1")
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(DEPS REQUIRED
            libgeneral
            libinsn)
    target_include_directories(offsetfinder64 PRIVATE "${DEPS_INCLUDE_DIRS}")
endif()

execute_process(COMMAND git rev-list --count HEAD WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" OUTPUT_VARIABLE VERSION_COMMIT_COUNT ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(COMMAND git rev-parse HEAD WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" OUTPUT_VARIABLE VERSION_COMMIT_SHA ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(COMMAND cp "${CMAKE_SOURCE_DIR}/pkgconfig/liboffsetfinder64.pc.in" "${CMAKE_SOURCE_DIR}/pkgconfig/liboffsetfinder64.pc" WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(COMMAND sed -i "s/VERSION_COUNT/${VERSION_COMMIT_COUNT}/g" ${CMAKE_SOURCE_DIR}/pkgconfig/liboffsetfinder64.pc WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
string(REPLACE "/lib" "" CMAKE_INSTALL_PREFIX2 ${CMAKE_INSTALL_PREFIX})
string(REPLACE "/" "\\/" CMAKE_INSTALL_PREFIX1 ${CMAKE_INSTALL_PREFIX2})
execute_process(COMMAND sed -i "s/PREFIX/${CMAKE_INSTALL_PREFIX1}/g" "${CMAKE_SOURCE_DIR}/pkgconfig/liboffsetfinder64.pc" WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" OUTPUT_STRIP_TRAILING_WHITESPACE)
add_definitions(
		-DVERSION_COMMIT_COUNT="${VERSION_COMMIT_COUNT}"
		-DVERSION_COMMIT_SHA="${VERSION_COMMIT_SHA}")
install(FILES pkgconfig/liboffsetfinder64.pc
		DESTINATION "${CMAKE_INSTALL_PREFIX}/pkgconfig")
install(FILES
		include/liboffsetfinder64/common.h
		include/liboffsetfinder64/ibootpatchfinder64.hpp
		include/liboffsetfinder64/ibootpatchfinder64_base.hpp
		include/liboffsetfinder64/ibootpatchfinder64_iOS14.hpp
		include/liboffsetfinder64/kernelpatchfinder64.hpp
		include/liboffsetfinder64/machopatchfinder64.hpp
		include/liboffsetfinder64/OFexception.hpp
		include/liboffsetfinder64/patch.hpp
		include/liboffsetfinder64/patchfinder64.hpp
		DESTINATION "${CMAKE_INSTALL_PREFIX1}/include/liboffsetfinder64")
install(TARGETS offsetfinder64
        DESTINATION ${CMAKE_INSTALL_PREFIX})
