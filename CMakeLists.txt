cmake_minimum_required(VERSION 3.1)
project(bgfx)

if(APPLE)
    include_directories("bx/include/compat/osx")
elseif(${CMAKE_SYSTEM_NAME} STREQUAL "FreeBSD")
    include_directories("bx/include/compat/freebsd")
endif()

# -------------------- 3rd party -----------------------
include_directories(3rdparty/khronos)

file(GLOB BGFX_3RD_PARTY
    "./bimg/src/*.cpp"
    "./bimg/3rdparty/edtaa3/*.cpp"
    "./bimg/3rdparty/etc1/*.cpp"
    "./bimg/3rdparty/etc2/*.cpp"
    "./bimg/3rdparty/iqa/source/*.cpp"
    "./bgfx/3rdparty/fcpp/*.cpp"
    "./bgfx/3rdparty/ib-compress/*.cpp"
    "./bimg/3rdparty/libsquish/*.cpp"
    "./bgfx/3rdparty/ocornut-imgui/*.cpp"
    "./bgfx/3rdparty/stb/*.cpp"
    )

add_library(bgfx3rdParty STATIC ${BGFX_3RD_PARTY})

set_target_properties(bgfx3rdParty PROPERTIES LINKER_LANGUAGE C)

target_include_directories(bgfx3rdParty PUBLIC bgfx/3rdparty/khronos)
target_include_directories(bgfx3rdParty PUBLIC bgfx/3rdparty)
target_include_directories(bgfx3rdParty PUBLIC bx/include)
target_include_directories(bgfx3rdParty PUBLIC bimg/include)
target_include_directories(bgfx3rdParty PUBLIC bimg/3rdparty)
target_include_directories(bgfx3rdParty PUBLIC bimg/3rdparty/iqa/include)

if(MINGW)
    target_include_directories(bgfx3rdParty PUBLIC bx/include/compat/mingw)

    add_definitions(-DBGFX_CONFIG_RENDERER_DIRECT3D9=0)
    add_definitions(-DBGFX_CONFIG_RENDERER_DIRECT3D11=0)
    add_definitions(-DBGFX_CONFIG_RENDERER_DIRECT3D12=0)
    add_definitions(-D__STDC_FORMAT_MACROS)
    

    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wunused-value")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fdata-sections")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ffunction-sections")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse2")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wunused-value")
    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wundef")
endif()

if(MSVC)
    add_definitions(-DBGFX_CONFIG_RENDERER_OPENGL=1)
    add_definitions(-DBGFX_CONFIG_RENDERER_DIRECT3D11=0)

	target_include_directories(bgfx3rdParty PUBLIC bx/include/compat/msvc)
	add_definitions(-D_CRT_SECURE_NO_WARNINGS)
	add_definitions(-D__STDC_FORMAT_MACROS)
	
	SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHsc /wd4577 /wd4996")
	
	target_link_libraries(bgfx3rdParty Psapi)
endif()

# ------------------ Main library ----------------------
SET(BX_SRC "./bx/src/amalgamated.cpp")
SET(BGFX_SRC "./bgfx/src/amalgamated.cpp")

if(APPLE)
    list(APPEND BGFX_SRC
        "./bgfx/src/glcontext_nsgl.mm"
        "./bgfx/src/renderer_mtl.mm"
    )
endif()

add_library(bx STATIC ${BX_SRC})

add_library(bgfx STATIC ${BGFX_SRC})
set_target_properties(bgfx PROPERTIES LINKER_LANGUAGE C)

if(MINGW)
    target_include_directories(bx PUBLIC bx/include/compat/mingw)
endif()

if(MSVC)
    target_include_directories(bx PUBLIC bx/include/compat/msvc)
endif()

if(APPLE)
    set_target_properties(bgfx PROPERTIES CXX_STANDARD 98)

    find_library(COCOA_LIBRARY Cocoa)
    if(NOT COCOA_LIBRARY)
        message(FATAL_ERROR "Could not find Cocoa library")
    else()
        message(STATUS "Found Cocoa library: ${COCOA_LIBRARY}")
    endif()

    find_library(METAL_LIBRARY Metal)
    if(METAL_LIBRARY)
        message(STATUS "Found Metal library: ${METAL_LIBRARY}")
    endif()

    find_library(QUARTZ_LIBRARY QuartzCore)
    if(QUARTZ_LIBRARY)
        message(STATUS "Found Quartz library: ${QUARTZ_LIBRARY}")
    endif()

    find_package(OpenGL REQUIRED)

    list(APPEND OPENGL_LIBRARIES "${COCOA_LIBRARY}" "${METAL_LIBRARY}" "${QUARTZ_LIBRARY}")
else()
    if(NOT ANDROID)
        find_package(OpenGL REQUIRED)
    endif()
endif()

if(ANDROID)
    target_link_libraries(bgfx log dl)
elseif(UNIX)
    set(DL "")
    if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
        set(DL dl)
    endif()

    find_package(X11 REQUIRED)

    target_link_libraries(bgfx ${X11_LIBRARIES} ${OPENGL_LIBRARIES} pthread ${DL})
    target_link_libraries(bx pthread ${DL})
endif()

target_include_directories(bx PUBLIC bx/include)
target_include_directories(bx PUBLIC bx/3rdparty)

target_include_directories(bgfx PUBLIC bgfx/include)
target_include_directories(bgfx PUBLIC bx/include)
target_link_libraries(bgfx bgfx3rdParty bx)
    
if(APPLE)
    target_include_directories(bgfx PRIVATE ${MALLOC_INCLUDE_DIR})
endif()

target_include_directories(bgfx PRIVATE ${OPENGL_INCLUDE_DIR})

# ------------------ Common library ----------------------
file(GLOB COMMON_SRC
    "./bgfx/examples/common/bounds.cpp"
    "./bgfx/examples/common/cube_atlas.cpp"
    "./bgfx/examples/common/*.cpp"
    "./bgfx/examples/common/debugdraw/*.cpp"
    "./bgfx/examples/common/entry/*.cpp"
    "./bgfx/examples/common/font/*.cpp"
    "./bgfx/examples/common/imgui/*.cpp"
    "./bgfx/examples/common/nanovg/*.cpp")
    

if(APPLE)
    list(APPEND COMMON_SRC "./bgfx/examples/common/entry/entry_osx.mm")
endif()

add_library(bgfx_common STATIC ${COMMON_SRC})
target_link_libraries(bgfx_common bgfx bx bgfx3rdParty)
set_target_properties(bgfx_common PROPERTIES LINKER_LANGUAGE C)

target_include_directories(bgfx_common PUBLIC bgfx/examples/common)
target_include_directories(bgfx_common PUBLIC bgfx/examples)

if(APPLE)
    target_include_directories(bgfx_common PRIVATE ${MALLOC_INCLUDE_DIR})
elseif(UNIX)
    target_include_directories(bgfx_common PRIVATE ${X11_INCLUDE_DIR}) # entry_x11.cpp
endif()

target_compile_definitions(bgfx PUBLIC BGFX_CONFIG_DEBUG=${BGFX_DEBUG})
target_compile_definitions(bx PUBLIC BGFX_CONFIG_DEBUG=${BGFX_DEBUG})
