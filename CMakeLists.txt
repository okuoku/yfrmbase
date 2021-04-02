set(YFRM_CWGL_USE_VULKAN OFF)
if("${YFRM_CWGL_BACKEND}" STREQUAL PlatformGLES)
    set(YFRM_CWGL_USE_ANGLE OFF)
else()
    set(YFRM_CWGL_USE_ANGLE ON)
endif()
if(ANDROID)
    set(YFRM_FS_USE_SDL ON)
else()
    set(YFRM_FS_USE_CXX17 ON)
endif()

cmake_minimum_required(VERSION 3.0)
project(yfrm C CXX)

# Configure SPIR-V dependencies
if(YFRM_CWGL_USE_VULKAN)
    set(SPIRV_WERROR OFF CACHE BOOL "" FORCE)
    set(SPIRV-Headers_SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/SPIRV-Headers)
    add_subdirectory(ext/cwgl/glslang glslang)
    add_subdirectory(ext/cwgl/SPIRV-Tools SPIRV-Tools)

    set(ANGLE_BACKEND Vulkan)
    set(ANGLE_SPIRV_HEADERS_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/SPIRV-Headers)
    set(ANGLE_VULKAN_HEADERS_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/Vulkan-Headers)
    set(ANGLE_SPIRV_TOOLS_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/SPIRV-Tools)
    set(ANGLE_VOLK_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/volk)
    set(ANGLE_VMA_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/VulkanMemoryAllocator)
    set(ANGLE_GLSLANG_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/glslang)

    set(backend_libs
        SPIRV-Tools
        glslang
        glslang-default-resource-limits
        SPIRV
        )
endif()

# Configure ANGLE
if(YFRM_CWGL_USE_ANGLE)
    set(ANGLE_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/angle)
    set(CHROMIUM_ZLIB_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/chromium-zlib)
    add_subdirectory(ext/cwgl/angle-static-cmake angle)
endif()

# Configure SDL
set(SDL_SHARED OFF CACHE BOOL "" FORCE)
set(SDL_STATIC ON CACHE BOOL "" FORCE)
add_subdirectory(ext/platform/SDL sdl)
target_compile_definitions(SDL2-static
    PUBLIC
    IID_IDXGIFactory2=SDL_IID_IDXGIFactory2)

include_directories(ext/platform/SDL/include)
include_directories(ext/cwgl/angle/include) # also provides GLES header
include_directories(yuniframe/include)
include_directories(ext/test/imgui)

set(CMAKE_CXX_STANDARD 17)

add_definitions(
    # Use static-link
    -DANGLE_EXPORT=
    -DEGLAPI=
    -DGL_API=
    -DGL_APIENTRY=
    -DGL_APICALL=
    )

set(yfrmsrc
    yuniframe/cwgl/src-gles2/cwgl-gles2-objstr.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-priv.h
    yuniframe/cwgl/src-gles2/cwgl-gles2-s2.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-s3.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-s4.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-s5.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-s6obj.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-s6shader.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-snone.c
    yuniframe/platform/src-sdl2/yfrm-audio-sdl2.c
    yuniframe/platform/src-sdl2/yfrm-ctx-sdl2.c
    yuniframe/platform/src-sdl2/yfrm-events-sdl2.c
    yuniframe/platform/src-sdl2/yfrm-main-sdl2.c
    )

if(YFRM_FS_USE_SDL)
    list(APPEND yfrmsrc
        yuniframe/platform/src-sdl2/yfrm-fs-sdl2.c
        )
elseif(YFRM_FS_USE_CXX17)
    list(APPEND yfrmsrc
        yuniframe/platform/src-cxx17/yfrm-fs-cxx17.cpp
        )
endif()

if(YFRM_CWGL_USE_VULKAN)
    add_definitions(-DYFRM_CWGL_USE_VULKAN)
else()
endif()

if(WIN32 OR WINDOWS_STORE)
    add_definitions(-DYFRM_CWGL_USE_DX11)
    list(APPEND yfrmsrc
        yuniframe/platform/src-win/yfrm-gpu-d3d11.c
        )
endif()

if(YFRM_CWGL_USE_ANGLE)
    add_definitions(-DYFRM_CWGL_USE_ANGLE)
    list(APPEND yfrmsrc
        yuniframe/platform/src-angle/yfrm-cwgl-angle.c
        )
endif()

set(sdlmain)
if(WINDOWS_STORE)
    add_definitions(-DYFRM_USE_UWP)
    set(sdlmain
        ext/platform/SDL/src/main/winrt/SDL_winrt_main_NonXAML.cpp)
    set_source_files_properties(
        ${sdlmain}
        PROPERTIES
        COMPILE_FLAGS /ZW
        )
elseif(WIN32)
    set(sdlmain
        ext/platform/SDL/src/main/windows/SDL_windows_main.c)
endif()

if(NOT YFRM_CWGL_BACKEND)
    add_executable(fwtest WIN32
        ${yfrmsrc}
        ${sdlmain}
        yuniframe/imgui/imgui_impl_cwgl.cpp
        yuniframe/imgui/imgui_impl_yfrm.cpp
        yuniframe/testimgui.cpp
        ext/test/imgui/imgui.cpp
        ext/test/imgui/imgui_demo.cpp
        ext/test/imgui/imgui_draw.cpp
        ext/test/imgui/imgui_tables.cpp
        ext/test/imgui/imgui_widgets.cpp
        )

    target_link_libraries(fwtest
        angle_static
        SDL2-static

        # SDL2
        user32
        gdi32
        winmm
        imm32
        ole32
        oleaut32
        version
        uuid
        advapi32
        setupapi
        shell32
        dinput8

        # angle
        dxgi
        dxguid

        # dx11
        d3d11

        ${backend_libs}
        )
endif()

add_library(yfrm OBJECT
    ${yfrmsrc}
    ${sdlmain})
