cmake_minimum_required(VERSION 3.0)
project(yfrm C CXX)

# Configure ANGLE
set(ANGLE_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/angle)
set(CHROMIUM_ZLIB_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/chromium-zlib)
add_subdirectory(ext/cwgl/angle-static-cmake angle)

# Configure SDL
set(SDL_SHARED OFF CACHE BOOL "" FORCE)
set(SDL_STATIC ON CACHE BOOL "" FORCE)
add_subdirectory(ext/platform/SDL sdl)
target_compile_definitions(SDL2-static
    PUBLIC
    IID_IDXGIFactory2=SDL_IID_IDXGIFactory2)

include_directories(ext/platform/SDL/include)
include_directories(ext/cwgl/angle/include)
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

    # Use DX11
    -DYFRM_CWGL_USE_DX11
    )

if(WINDOWS_STORE)
    add_definitions(-DYFRM_USE_UWP)
endif()

add_executable(fwtest
    yuniframe/cwgl/src-gles2/cwgl-gles2-objstr.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-priv.h
    yuniframe/cwgl/src-gles2/cwgl-gles2-s2.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-s3.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-s4.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-s5.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-s6obj.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-s6shader.c
    yuniframe/cwgl/src-gles2/cwgl-gles2-snone.c
    yuniframe/platform/src-cxx17/yfrm-fs-cxx17.cpp
    yuniframe/platform/src-sdl2/yfrm-audio-sdl2.c
    yuniframe/platform/src-sdl2/yfrm-ctx-sdl2.c
    yuniframe/platform/src-sdl2/yfrm-events-sdl2.c
    yuniframe/platform/src-angle/yfrm-cwgl-angle.c
    yuniframe/platform/src-win/yfrm-gpu-d3d11.c
    yuniframe/imgui/imgui_impl_cwgl.cpp
    yuniframe/imgui/imgui_impl_yfrm.cpp
    yuniframe/testimgui.cpp
    ext/test/imgui/imgui.cpp
    ext/test/imgui/imgui_demo.cpp
    ext/test/imgui/imgui_draw.cpp
    ext/test/imgui/imgui_tables.cpp
    ext/test/imgui/imgui_widgets.cpp

    main.c
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
    )
