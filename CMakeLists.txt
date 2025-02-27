cmake_minimum_required(VERSION 3.22)
set(CMAKE_C_FLAGS_RELEASE "-Os -DNDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE "-Os -DNDEBUG")
set(CMAKE_C_FLAGS_DEBUG "-g -O0 -DDEBUG")
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -DDEBUG")
set(CMAKE_C_STANDARD 17)
set(CMAKE_CXX_STANDARD 20)
set(ignoreMe "${NO_PKGCFG}")
project(iBoot64Patcher)
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
add_executable(iBoot64Patcher
		src/main.cpp)
target_include_directories(iBoot64Patcher PRIVATE
        "${CMAKE_SOURCE_DIR}/dep_root/include"
        src)
target_link_directories(iBoot64Patcher PRIVATE
        "${CMAKE_SOURCE_DIR}/dep_root/lib")
if(NOT DEFINED NO_PKGCFG AND NOT "$ENV{NO_PKGCFG}" MATCHES "1")
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(DEPS REQUIRED
            libgeneral
            libinsn
			liboffsetfinder64)
    target_include_directories(iBoot64Patcher PRIVATE "${DEPS_INCLUDE_DIRS}")
	target_link_directories(iBoot64Patcher PRIVATE "${DEPS_LIBRARY_DIRS}")
	target_link_libraries(futurerestore PRIVATE "${DEPS_LIBRARIES}"
			"-lgeneral"
			insn
			offsetfinder64)
else()
	target_include_directories(iBoot64Patcher PRIVATE "${CMAKE_SOURCE_DIR}/dep_root/include")
	target_link_directories(iBoot64Patcher PRIVATE "${CMAKE_SOURCE_DIR}/dep_root/lib")
	target_link_libraries(iBoot64Patcher PRIVATE
			"-lgeneral"
			insn
			offsetfinder64)
endif()

execute_process(COMMAND git rev-list --count HEAD WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" OUTPUT_VARIABLE VERSION_COMMIT_COUNT ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
execute_process(COMMAND git rev-parse HEAD WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}" OUTPUT_VARIABLE VERSION_COMMIT_SHA ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
add_definitions(
		-DVERSION_COMMIT_COUNT="${VERSION_COMMIT_COUNT}"
		-DVERSION_COMMIT_SHA="${VERSION_COMMIT_SHA}")
install(TARGETS iBoot64Patcher
        DESTINATION ${CMAKE_INSTALL_PREFIX}/bin)
