## initialize
# mkdir build && cd build
# cmake --preset [win32|win64]
#
# ninja
# ninja install
# ninja clean
# cpack
#
cmake_minimum_required(VERSION 3.15)
project(vkd3d-proton VERSION 2.4.0 LANGUAGES C CXX)

find_program(CCACHE ccache)
if (CCACHE)
  message(STATUS "found ccache: ${CCACHE}")
  set(CMAKE_C_COMPILER_LAUNCHER ccache)
endif()

foreach(header IN ITEMS
  include/vkd3d_d3d12.idl
  include/vkd3d_d3d12sdklayers.idl
  include/vkd3d_d3dcommon.idl
  include/vkd3d_dxgi.idl
  include/vkd3d_dxgi1_2.idl
  include/vkd3d_dxgi1_3.idl
  include/vkd3d_dxgi1_4.idl
  include/vkd3d_dxgibase.idl
  include/vkd3d_dxgiformat.idl
  include/vkd3d_dxgitype.idl
  include/vkd3d_swapchain_factory.idl
  )
  get_filename_component(header_name "${header}" NAME_WE)
  add_custom_command(
    OUTPUT "include/${header_name}.h"
    COMMAND widl -h -o "${PROJECT_BINARY_DIR}/include/${header_name}.h" "${PROJECT_SOURCE_DIR}/${header}"
    DEPENDS "${header}"
  )
  list(APPEND vkd3d_headers "include/${header_name}.h")
endforeach()

add_library(vkd3d_deps INTERFACE)
target_link_libraries(vkd3d_deps
  INTERFACE
  -Wl,--enable-stdcall-fixup
  -fvisibility=hidden
  -static-libgcc
  -static
)
target_compile_definitions(vkd3d_deps INTERFACE)
target_include_directories(vkd3d_deps
  INTERFACE
  "${PROJECT_SOURCE_DIR}/include"
  "${PROJECT_SOURCE_DIR}/subprojects/Vulkan-Headers/include"
  "${PROJECT_SOURCE_DIR}/subprojects/SPIRV-Headers/include"
  "${PROJECT_SOURCE_DIR}/include/private"
  "${PROJECT_BINARY_DIR}/include"
)
if(CMAKE_SIZEOF_VOID_P EQUAL 4)
  target_compile_options(vkd3d_deps INTERFACE -msse2 -msse)
endif()

add_library(vkd3d-common
  STATIC
  libs/vkd3d-common/debug.c
  libs/vkd3d-common/memory.c
  libs/vkd3d-common/string.c
  libs/vkd3d-common/utf8.c
)

target_link_libraries(vkd3d-common
  PRIVATE
  vkd3d_deps
)

option(DXIL_SPIRV_CLI "" ON)
add_subdirectory(subprojects/dxil-spirv EXCLUDE_FROM_ALL)

add_library(vkd3d-shader
  STATIC
  ${vkd3d_headers}
  libs/vkd3d-shader/checksum.c
  libs/vkd3d-shader/dxbc.c
  libs/vkd3d-shader/dxil.c
  libs/vkd3d-shader/spirv.c
  libs/vkd3d-shader/trace.c
  libs/vkd3d-shader/vkd3d_shader_main.c
)

target_link_libraries(vkd3d-shader PRIVATE vkd3d_deps vkd3d-common dxil-spirv-c-static)
target_compile_definitions(vkd3d-shader PRIVATE -DHAVE_DXIL_SPV)
target_include_directories(vkd3d-shader PRIVATE
  subprojects/dxil-spirv
)

execute_process(COMMAND git describe --always --tags --dirty=+
  OUTPUT_VARIABLE VCS_TAG
  OUTPUT_STRIP_TRAILING_WHITESPACE
  WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
)
configure_file("vkd3d_version.h.in" "${PROJECT_BINARY_DIR}/include/vkd3d_version.h")
execute_process(COMMAND git describe --always --exclude=* --abbrev=15 --dirty=0
  OUTPUT_VARIABLE VCS_TAG
  OUTPUT_STRIP_TRAILING_WHITESPACE
  WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
)
configure_file("vkd3d_build.h.in" "${PROJECT_BINARY_DIR}/include/vkd3d_build.h")

include(cmake/glsl.cmake)

find_program(glsl_compiler
  NAMES glslangValidator
  PATHS "$ENV{VK_SDK_PATH}/Bin"
  REQUIRED)

set_compiled_shaders(vkd3d_shaders
  libs/vkd3d/shaders/cs_clear_uav_buffer_float.comp
  libs/vkd3d/shaders/cs_clear_uav_buffer_raw.comp
  libs/vkd3d/shaders/cs_clear_uav_buffer_uint.comp
  libs/vkd3d/shaders/cs_clear_uav_image_1d_array_float.comp
  libs/vkd3d/shaders/cs_clear_uav_image_1d_array_uint.comp
  libs/vkd3d/shaders/cs_clear_uav_image_1d_float.comp
  libs/vkd3d/shaders/cs_clear_uav_image_1d_uint.comp
  libs/vkd3d/shaders/cs_clear_uav_image_2d_array_float.comp
  libs/vkd3d/shaders/cs_clear_uav_image_2d_array_uint.comp
  libs/vkd3d/shaders/cs_clear_uav_image_2d_float.comp
  libs/vkd3d/shaders/cs_clear_uav_image_2d_uint.comp
  libs/vkd3d/shaders/cs_clear_uav_image_3d_float.comp
  libs/vkd3d/shaders/cs_clear_uav_image_3d_uint.comp
  libs/vkd3d/shaders/cs_predicate_command.comp
  libs/vkd3d/shaders/cs_resolve_binary_queries.comp
  libs/vkd3d/shaders/cs_resolve_predicate.comp
  libs/vkd3d/shaders/cs_resolve_query.comp
  libs/vkd3d/shaders/fs_copy_image_float.frag
  libs/vkd3d/shaders/fs_copy_image_uint.frag
  libs/vkd3d/shaders/fs_swapchain_fullscreen.frag
  libs/vkd3d/shaders/gs_fullscreen.geom
  libs/vkd3d/shaders/vs_fullscreen_layer.vert
  libs/vkd3d/shaders/vs_fullscreen.vert
  libs/vkd3d/shaders/vs_swapchain_fullscreen.vert
)

add_library(vkd3d
  STATIC
  ${vkd3d_shaders}
  ${vkd3d_headers}
  libs/vkd3d/acceleration_structure.c
  libs/vkd3d/bundle.c
  libs/vkd3d/cache.c
  libs/vkd3d/command.c
  libs/vkd3d/debug_ring.c
  libs/vkd3d/device.c
  libs/vkd3d/heap.c
  libs/vkd3d/memory.c
  libs/vkd3d/meta.c
  libs/vkd3d/platform.c
  libs/vkd3d/raytracing_pipeline.c
  libs/vkd3d/resource.c
  libs/vkd3d/state.c
  libs/vkd3d/swapchain.c
  libs/vkd3d/utils.c
  libs/vkd3d/va_map.c
  libs/vkd3d/vkd3d_main.c
)

target_link_libraries(vkd3d PRIVATE vkd3d_deps vkd3d-common vkd3d-shader)
target_compile_definitions(vkd3d PRIVATE -DPACKAGE_VERSION="${PROJECT_VERSION}" -DVKD3D_BUILD_STANDALONE_D3D12)

add_library(d3d12
  SHARED
  libs/d3d12/main.c
  libs/d3d12/d3d12.def
)

target_link_libraries(d3d12 PRIVATE vkd3d_deps vkd3d -ldxgi)
set_property(TARGET d3d12 PROPERTY PREFIX "")
target_compile_definitions(d3d12 PRIVATE -DVKD3D_BUILD_STANDALONE_D3D12)

option(LTO "enable LTO" ON)

if(LTO)
  include(CheckIPOSupported)
  check_ipo_supported()
  set_property(TARGET d3d12 PROPERTY INTERPROCEDURAL_OPTIMIZATION TRUE)
endif()

if(NOT MSVC)
  find_program(cv2pdb cv2pdb)
  if(cv2pdb)
    message(STATUS "\"cv2pdb\" found: target \"pdb\" will be available")
    include(cmake/cv2pdb.cmake)
    add_cv2pdb_target(d3d12 d3d12.dll d3d12.pdb)
    add_custom_target(pdb DEPENDS d3d12_pdb)
  endif()
endif()

install(
  TARGETS d3d12
  RUNTIME
  DESTINATION "$<IF:$<EQUAL:${CMAKE_SIZEOF_VOID_P},8>,x64,x32>"
)

set(CPACK_GENERATOR ZIP)
include(cpack)

option(enable_extras "enable extras" OFF)

if(enable_extras)
  add_executable(gears
    WIN32
    demos/gears.c
    ${vkd3d_headers}
  )
  target_link_libraries(gears PRIVATE vkd3d_deps dxgi d3d12)

  add_executable(triangle
    WIN32
    demos/triangle.c
    ${vkd3d_headers}
  )
  target_link_libraries(triangle PRIVATE vkd3d_deps dxgi d3d12)
endif()

option(enable_tests "enable tests" OFF)

if(enable_tests)
  add_executable(d3d12_exe tests/d3d12.c)
  target_link_libraries(d3d12_exe vkd3d_deps dxgi d3d12 vkd3d-common)
  set_property(TARGET d3d12_exe PROPERTY OUTPUT_NAME "d3d12")

  add_executable(descriptor-performance tests/descriptor_performance.c)
  target_link_libraries(descriptor-performance vkd3d_deps dxgi d3d12)
endif()
