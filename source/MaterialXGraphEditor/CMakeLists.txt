set(DEAR_IMGUI_PREFIX "${CMAKE_CURRENT_SOURCE_DIR}/External/ImGui" CACHE STRING "Path to Dear ImGui")

if (NOT IS_DIRECTORY "${DEAR_IMGUI_PREFIX}/backends")
    message(FATAL_ERROR "Building the MaterialX graph editor requires the ImGui submodule "
        "to be present. Update your repository by calling the following:\n"
        "git submodule update --init --recursive")
endif()

if(MSVC)
    add_compile_options(-wd4100 -wd4152 -wd4201 -wd4244 -wd4456)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    add_compile_options(-Wno-unused -Wno-deprecated -Wno-comment -Wno-unused-variable -Wno-unused-parameter -Wno-shadow)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
    add_compile_options(-Wno-format-truncation -Wno-use-after-free -Wno-comment -Wno-unused-but-set-variable -Wno-unused-parameter -Wno-shadow)
endif()

set(MATERIALX_GRAPHEDITOR_RENDER_BACKEND_DEFINITIONS "")

if(APPLE)
    option(MATERIALX_GRAPHEDITOR_USE_METAL_BACKEND "Use Metal Backend for MaterialXGraphEditor on Apple Platforms" OFF)

    # Disables MaterialXGraphEditor Metal Rendering on MacOS version below 10.14
    if (CMAKE_HOST_SYSTEM_NAME STREQUAL "Darwin" AND CMAKE_HOST_SYSTEM_VERSION VERSION_LESS 19)
        set(MATERIALX_GRAPHEDITOR_USE_METAL_BACKEND OFF)
    endif()

    if(MATERIALX_GRAPHEDITOR_USE_METAL_BACKEND)
        set(MATERIALX_GRAPHEDITOR_RENDER_BACKEND_DEFINITIONS "-DMATERIALX_GRAPHEDITOR_METAL_BACKEND=1")
    else()
        set(MATERIALX_GRAPHEDITOR_RENDER_BACKEND_DEFINITIONS "-DMATERIALX_GRAPHEDITOR_OPENGL_BACKEND=1")
    endif()
endif()

file(GLOB materialx_source "${CMAKE_CURRENT_SOURCE_DIR}/*.cpp")
file(GLOB materialx_headers "${CMAKE_CURRENT_SOURCE_DIR}/*.h*")

if (APPLE)
    list(APPEND materialx_source ${CMAKE_CURRENT_SOURCE_DIR}/FileDialog_Darwin.mm)
endif()

if (MATERIALX_GRAPHEDITOR_USE_METAL_BACKEND)
    # Compile all source files as Objective-C++ so we can use Metal and other 
    # Obj-C APIs from within .cpp files.
    set_source_files_properties(${materialx_source} PROPERTIES LANGUAGE OBJCXX)
endif ()

file(GLOB imgui_source "${DEAR_IMGUI_PREFIX}/*.cpp")
file(GLOB imgui_headers "${DEAR_IMGUI_PREFIX}/*.h*")

if(MATERIALX_GRAPHEDITOR_USE_METAL_BACKEND)
set(imgui_source ${imgui_source}
    "${DEAR_IMGUI_PREFIX}/backends/imgui_impl_glfw.cpp"
    "${DEAR_IMGUI_PREFIX}/backends/imgui_impl_metal.mm"
    "${DEAR_IMGUI_PREFIX}/misc/cpp/imgui_stdlib.cpp")
set(imgui_headers ${imgui_headers}
    "${DEAR_IMGUI_PREFIX}/backends/imgui_impl_glfw.h"
    "${DEAR_IMGUI_PREFIX}/backends/imgui_impl_metal.h"
    "${DEAR_IMGUI_PREFIX}/misc/cpp/imgui_stdlib.h")
else()
set(imgui_source ${imgui_source}
    "${DEAR_IMGUI_PREFIX}/backends/imgui_impl_glfw.cpp"
    "${DEAR_IMGUI_PREFIX}/backends/imgui_impl_opengl3.cpp"
    "${DEAR_IMGUI_PREFIX}/misc/cpp/imgui_stdlib.cpp")
set(imgui_headers ${imgui_headers}
    "${DEAR_IMGUI_PREFIX}/backends/imgui_impl_glfw.h"
    "${DEAR_IMGUI_PREFIX}/backends/imgui_impl_opengl3.h"
    "${DEAR_IMGUI_PREFIX}/misc/cpp/imgui_stdlib.h")
endif()

file(GLOB imguinode_source "${CMAKE_CURRENT_SOURCE_DIR}/External/ImGuiNodeEditor/*.cpp")
file(GLOB imguinode_headers "${CMAKE_CURRENT_SOURCE_DIR}/External/ImGuiNodeEditor/*.h*")

set(imguinode_source ${imguinode_source}
    "${CMAKE_CURRENT_SOURCE_DIR}/External/ImGuiNodeEditor/examples/blueprints-example/utilities/drawing.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/External/ImGuiNodeEditor/examples/blueprints-example/utilities/widgets.cpp")
set(imguinode_headers ${imguinode_headers}
    "${CMAKE_CURRENT_SOURCE_DIR}/External/ImGuiNodeEditor/examples/blueprints-example/utilities/drawing.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/External/ImGuiNodeEditor/examples/blueprints-example/utilities/widgets.h")

assign_source_group("Source Files" ${materialx_source} ${imgui_source} ${imguinode_source})
assign_source_group("Header Files" ${materialx_headers} ${imgui_headers} ${imguinode_headers})

include_directories("${DEAR_IMGUI_PREFIX}")
include_directories("${DEAR_IMGUI_PREFIX}/backends")
include_directories("${DEAR_IMGUI_PREFIX}/misc/cpp")
include_directories("${CMAKE_CURRENT_SOURCE_DIR}/External/ImGuiNodeEditor")
include_directories("${CMAKE_CURRENT_SOURCE_DIR}/External/ImGuiNodeEditor/examples/blueprints-example/utilities")

set(GLFW_BUILD_EXAMPLES OFF)
set(GLFW_BUILD_TESTS OFF)
set(GLFW_BUILD_DOCS OFF)
set(GLFW_INSTALL OFF)

# Explicitly build GLFW with static libraries, independent of the broader build settings.
set(BUILD_SHARED_LIBS OFF)

add_subdirectory(External/Glfw)
set_property(TARGET glfw_minimal PROPERTY FOLDER "External")

if(MSVC)
    SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /ENTRY:mainCRTStartup")
endif()

add_definitions(${MATERIALX_GRAPHEDITOR_RENDER_BACKEND_DEFINITIONS})

add_executable(MaterialXGraphEditor
    ${materialx_source}
    ${materialx_headers}
    ${imgui_source}
    ${imgui_headers}
    ${imguinode_source}
    ${imguinode_headers})

if(APPLE)
    target_compile_options(MaterialXGraphEditor PRIVATE -fobjc-arc)
endif()

if(MATERIALX_GRAPHEDITOR_USE_METAL_BACKEND)
set(MATERIALX_LIBRARIES
    MaterialXFormat
    MaterialXGenMsl
    MaterialXRenderMsl)
else()
set(MATERIALX_LIBRARIES
    MaterialXFormat
    MaterialXGenGlsl
    MaterialXRenderGlsl)
endif()

if(APPLE)
    find_library(CORE_FOUNDATION Foundation REQUIRED)
    list(APPEND MATERIALX_LIBRARIES ${CORE_FOUNDATION})
    if(MATERIALX_GRAPHEDITOR_USE_METAL_BACKEND)
        find_library(METAL Metal REQUIRED)
        find_library(QUARTZ_CORE QuartzCore REQUIRED)
        list(APPEND MATERIALX_LIBRARIES ${METAL} ${QUARTZ_CORE})
    endif()
endif()

target_link_libraries(
    MaterialXGraphEditor
    PRIVATE
    ${MATERIALX_LIBRARIES}
 	glfw_minimal)

install(TARGETS MaterialXGraphEditor
    EXPORT MaterialX
    RUNTIME DESTINATION ${MATERIALX_INSTALL_BIN_PATH})
if(MSVC)
    install(FILES $<TARGET_PDB_FILE:MaterialXGraphEditor>
            DESTINATION ${MATERIALX_INSTALL_BIN_PATH} OPTIONAL)
endif()
