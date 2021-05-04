if("${YFRM_CWGL_BACKEND}" STREQUAL PlatformGLES)
    set(YFRM_CWGL_USE_ANGLE OFF)
else()
    set(YFRM_CWGL_USE_ANGLE ON)
endif()
if(ANDROID OR APPLE)
    set(YFRM_FS_USE_SDL ON)
else()
    set(YFRM_FS_USE_CXX17 ON)
endif()

if(YFRM_CWGL_EXPERIMENTAL_TRACKER)
    set(YFRM_CWGL_USE_ANGLE OFF)
endif()

cmake_minimum_required(VERSION 3.0)
project(yfrm C CXX)

# Configure SPIR-V dependencies
if(YFRM_CWGL_USE_VULKAN OR YFRM_CWGL_USE_METAL)
    set(SPIRV_WERROR OFF CACHE BOOL "" FORCE)
    set(SPIRV-Headers_SOURCE_DIR ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/SPIRV-Headers)
    set(SKIP_GLSLANG_INSTALL OFF CACHE BOOL "" FORCE)
    set(ENABLE_GLSLANG_BINARIES OFF CACHE BOOL "" FORCE)
    add_subdirectory(ext/cwgl/glslang glslang)
    # Workaround for glslang-default-resource-limits
    set(glslang-resource-limits-srcs
        ext/cwgl/glslang/StandAlone/ResourceLimits.cpp
        ext/cwgl/glslang/StandAlone/resource_limits_c.cpp)
    add_library(my-glslang-default-resource-limits STATIC
        ${glslang-resource-limits-srcs})

    set(SKIP_SPIRV_TOOLS_INSTALL OFF CACHE BOOL "" FORCE)
    set(SPIRV_SKIP_EXECUTABLES ON)
    add_subdirectory(ext/cwgl/SPIRV-Tools SPIRV-Tools)
    if(YFRM_CWGL_USE_METAL)
        set(SPIRV_CROSS_CLI OFF CACHE BOOL "" FORCE)
        set(SPIRV_CROSS_SKIP_INSTALL ON CACHE BOOL "" FORCE)
        add_subdirectory(ext/cwgl/SPIRV-Cross SPIRV-Cross)
    endif()

    set(ANGLE_SPIRV_HEADERS_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/SPIRV-Headers)
    set(ANGLE_VULKAN_HEADERS_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/Vulkan-Headers)
    set(ANGLE_SPIRV_TOOLS_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/SPIRV-Tools)
    set(ANGLE_VOLK_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/volk) # Vulkan only?
    set(ANGLE_VMA_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/VulkanMemoryAllocator) # Vulkan only?
    set(ANGLE_GLSLANG_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/glslang)
    set(ANGLE_SPIRV_CROSS_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/SPIRV-Cross)

    if(YFRM_CWGL_USE_METAL)
        set(ANGLE_BACKEND Metal)
        set(backend_libs spirv-cross-msl "-framework IOSurface")
    else()
        set(ANGLE_BACKEND Vulkan)
        set(backend_libs)
    endif()

    list(APPEND backend_libs
        SPIRV-Tools
        glslang
        my-glslang-default-resource-limits
        SPIRV
        )
endif()

# Configure ANGLE
if(YFRM_CWGL_USE_ANGLE)
    set(ANGLE_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/angle)
    set(CHROMIUM_ZLIB_ROOT ${CMAKE_CURRENT_LIST_DIR}/ext/cwgl/chromium-zlib)
    set(ANGLE_WORKAROUND_FAKEOBJCSOURCE ON)
    add_subdirectory(ext/cwgl/angle-static-cmake angle)
endif()

# Configure SDL
set(SDL_SHARED OFF CACHE BOOL "" FORCE)
set(SDL_STATIC ON CACHE BOOL "" FORCE)
set(LIBC ON CACHE BOOL "" FORCE)
if(APPLE)
    # FIXME: Metal detection do not work on upstream SDL
    #        we don't have to bothered with auto detection anyway, anyhow
    set(HAVE_FRAMEWORK_METAL ON CACHE BOOL "" FORCE)
endif()
add_subdirectory(ext/platform/SDL sdl)

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

if(YFRM_BUILD_MODULE)
    add_definitions(
        -DCWGL_DLL
        -DCWGL_SHARED_BUILD
        -DYFRM_DLL
        -DYFRM_SHARED_BUILD
        )
endif()

if(YFRM_CWGL_EXPERIMENTAL_TRACKER)
    set(cwglsrc
        yuniframe/cwgl/src-tracker/cwgl-tracker-objstr.c
        yuniframe/cwgl/src-tracker/cwgl-tracker-priv.h
        yuniframe/cwgl/src-tracker/cwgl-tracker-s2.c
        yuniframe/cwgl/src-tracker/cwgl-tracker-s3.c
        yuniframe/cwgl/src-tracker/cwgl-tracker-s4.c
        yuniframe/cwgl/src-tracker/cwgl-tracker-s5.c
        yuniframe/cwgl/src-tracker/cwgl-tracker-s6obj.c
        yuniframe/cwgl/src-tracker/cwgl-tracker-s6shader.c
        yuniframe/cwgl/src-tracker/cwgl-tracker-snone.c
        yuniframe/cwgl/src-tracker/cwgl-tracker-vao.c
        )
else()
    set(cwglsrc
        yuniframe/cwgl/src-gles2/cwgl-gles2-objstr.c
        yuniframe/cwgl/src-gles2/cwgl-gles2-priv.h
        yuniframe/cwgl/src-gles2/cwgl-gles2-s2.c
        yuniframe/cwgl/src-gles2/cwgl-gles2-s3.c
        yuniframe/cwgl/src-gles2/cwgl-gles2-s4.c
        yuniframe/cwgl/src-gles2/cwgl-gles2-s5.c
        yuniframe/cwgl/src-gles2/cwgl-gles2-s6obj.c
        yuniframe/cwgl/src-gles2/cwgl-gles2-s6shader.c
        yuniframe/cwgl/src-gles2/cwgl-gles2-snone.c
        yuniframe/cwgl/src-gles2/cwgl-gles2-vao.c
        )
endif()

set(yfrmsrc
    ${cwglsrc}
    yuniframe/platform/src-sdl2/yfrm-audio-sdl2.c
    yuniframe/platform/src-sdl2/yfrm-ctx-sdl2.c
    yuniframe/platform/src-sdl2/yfrm-events-sdl2.c
    )

if(NOT YFRM_BUILD_MODULE)
    list(APPEND yfrmsrc
        yuniframe/platform/src-sdl2/yfrm-main-sdl2.c
    )
endif()

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
elseif(WIN32 OR WINDOWS_STORE)
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
    list(APPEND yfrmsrc
        yuniframe/platform/src-win/yfrm-execpath-win32.c
        )
    set(sdlmain
        ext/platform/SDL/src/main/windows/SDL_windows_main.c)
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL iOS OR
        "${CMAKE_SYSTEM_NAME}" STREQUAL tvOS OR
        "${CMAKE_SYSTEM_NAME}" STREQUAL watchOS)
    # UIKit platforms
    set(sdlmain
        ext/platform/SDL/src/main/uikit/SDL_uikit_main.c
        )
endif()

if(WIN32 OR WINDOWS_STORE)
    set(platform_libs
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
endif()

if(NOT YFRM_CWGL_BACKEND)
    if(MSVC)
        if(YFRM_CWGL_USE_ANGLE)
            target_compile_options(angle_static PRIVATE /MP)
        endif()
        target_compile_options(SDL2-static PRIVATE /MP)
        add_compile_options(/MP)
    endif()

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

    if(YFRM_CWGL_USE_ANGLE)
        set(gles_libs angle_static)
    endif()

    target_link_libraries(fwtest
        ${gles_libs}
        SDL2-static

        ${platform_libs}

        ${backend_libs}
        )
endif()

if(YFRM_BUILD_MODULE)
    add_library(yfrm SHARED
        ${yfrmsrc})
    target_link_libraries(yfrm
        angle_static
        SDL2-static
        ${platform_libs}
        ${backend_libs}
        )
    if(YFRM_DEBUG_COPY)
        add_custom_command(TARGET yfrm POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy
            "$<TARGET_FILE:yfrm>"
            "${YFRM_DEBUG_COPY}"
            )
    endif()
else()
    add_library(yfrm OBJECT
        ${yfrmsrc}
        ${sdlmain})
endif()
