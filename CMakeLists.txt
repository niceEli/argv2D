# CMakeList.txt : CMake project for vulPlay, include source and define
# project specific logic here.
#
cmake_minimum_required (VERSION 3.8)

# Enable Hot Reload for MSVC compilers if supported.
if (POLICY CMP0141)
  cmake_policy(SET CMP0141 NEW)
  set(CMAKE_MSVC_DEBUG_INFORMATION_FORMAT "$<IF:$<AND:$<C_COMPILER_ID:MSVC>,$<CXX_COMPILER_ID:MSVC>>,$<$<CONFIG:Debug,RelWithDebInfo>:EditAndContinue>,$<$<CONFIG:Debug,RelWithDebInfo>:ProgramDatabase>>")
endif()

project ("argv")

# Add source to this project's executable.
file(GLOB_RECURSE src
      RELATIVE ${CMAKE_SOURCE_DIR}
      CONFIGURE_DEPENDS
      "./src/*.h"
      "./src/*.c"
      "./src/*.hpp"
      "./src/*.cpp"
)

list(REMOVE_ITEM src "./src/main.cpp")

add_executable (${PROJECT_NAME} "./src/main.cpp" ${src})

# Packages Start

find_package(fmt CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE fmt::fmt)

find_package(SDL2 CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME}
    PRIVATE
    $<TARGET_NAME_IF_EXISTS:SDL2::SDL2main>
    $<IF:$<TARGET_EXISTS:SDL2::SDL2>,SDL2::SDL2,SDL2::SDL2-static>
)

find_package(SDL2_image CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE $<IF:$<TARGET_EXISTS:SDL2_image::SDL2_image>,SDL2_image::SDL2_image,SDL2_image::SDL2_image-static>)

find_package(SDL2_mixer CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE $<IF:$<TARGET_EXISTS:SDL2_mixer::SDL2_mixer>,SDL2_mixer::SDL2_mixer,SDL2_mixer::SDL2_mixer-static>)

find_package(SDL2_ttf CONFIG REQUIRED)
target_link_libraries(${PROJECT_NAME} PRIVATE $<IF:$<TARGET_EXISTS:SDL2_ttf::SDL2_ttf>,SDL2_ttf::SDL2_ttf,SDL2_ttf::SDL2_ttf-static>)

# Packages End

if (CMAKE_VERSION VERSION_GREATER 3.12)
  set_property(TARGET ${PROJECT_NAME} PROPERTY CXX_STANDARD 23)
endif()

############## Build SHADERS #######################

# Find all vertex and fragment sources within shaders directory
# taken from VBlancos vulkan tutorial
# https://github.com/vblanco20-1/vulkan-guide/blob/all-chapters/CMakeLists.txt
find_program(GLSL_VALIDATOR glslangValidator HINTS 
  ${Vulkan_GLSLANG_VALIDATOR_EXECUTABLE} 
  /bin
  /usr/bin 
  /usr/local/bin 
  $ENV{VULKAN_SDK}/Bin/ 
  $ENV{VULKAN_SDK}/Bin32/
)

message(STATUS "glslangValidator: ${GLSL_VALIDATOR}")

# get all .vert and .frag files in shaders directory
file(GLOB_RECURSE GLSL_SOURCE_FILES
  CONFIGURE_DEPENDS
  "${PROJECT_SOURCE_DIR}/shaders/*.frag"
  "${PROJECT_SOURCE_DIR}/shaders/*.vert"
  "${PROJECT_SOURCE_DIR}/shaders/*.comp"
  "${PROJECT_SOURCE_DIR}/shaders/*.glsl"
)

message(STATUS "GLSL_SOURCE_FILES: ${GLSL_SOURCE_FILES}")

# Assume BUILT_FILES_DIR is defined elsewhere or set it here
set(BUILT_FILES_DIR "${CMAKE_BINARY_DIR}/shaders")

# Create output directory
file(MAKE_DIRECTORY ${BUILT_FILES_DIR})

# List to store SPIR-V files
set(SPIRV_BINARY_FILES "")

# Create custom commands and dependencies for each shader file
foreach(GLSL ${GLSL_SOURCE_FILES})
  get_filename_component(FILE_NAME ${GLSL} NAME)
  set(SPIRV "${BUILT_FILES_DIR}/${FILE_NAME}.spv")
  
  add_custom_command(
    OUTPUT ${SPIRV}
    COMMAND ${GLSL_VALIDATOR} ${GLSL} -V -o ${SPIRV}
    DEPENDS ${GLSL}
    COMMENT "Compiling ${GLSL} to ${SPIRV}"
    VERBATIM
  )

  list(APPEND SPIRV_BINARY_FILES ${SPIRV})
endforeach()

# Create a custom target that depends on all the SPIR-V files
add_custom_target(compile_shaders ALL DEPENDS ${SPIRV_BINARY_FILES})

# Ensure the main project depends on the compile_shaders target
add_dependencies(${PROJECT_NAME} compile_shaders)

message(STATUS "BUILT_SHA_FILES_DIR: ${BUILT_FILES_DIR}")

## end of build shaders

## add res folder to the built folder

## Copy resource folder and update on changes
set(RES_DIR "${CMAKE_SOURCE_DIR}/res")
set(BUILT_RES_DIR "${CMAKE_BINARY_DIR}/res")

add_custom_command(
  OUTPUT ${BUILT_RES_DIR}
  COMMAND ${CMAKE_COMMAND} -E copy_directory ${RES_DIR} ${BUILT_RES_DIR}
  DEPENDS ${RES_DIR}
  COMMENT "Copying resources to ${BUILT_RES_DIR}"
)

add_custom_target(copy_resources ALL DEPENDS ${BUILT_RES_DIR})
add_dependencies(${PROJECT_NAME} copy_resources)
