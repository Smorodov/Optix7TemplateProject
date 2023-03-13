cmake_minimum_required(VERSION 3.15)
project(Optix7TemplateProject LANGUAGES CXX CUDA)

set(CMAKE_BINARY_DIR ${CMAKE_SOURCE_DIR}/build)
set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR})
set(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR})

# OptiX_INCLUDE "Z:/OptiX SDK 7.1.0/include"
# Optix_INSTALL_DIR "Z:/OptiX SDK 7.1.0/SDK"

if (POLICY CMP0048)
  cmake_policy(SET CMP0048 NEW)
endif (POLICY CMP0048)

set (CUDA_PROPAGATE_HOST_FLAGS ON)
find_package(OpenGL REQUIRED)
# ------------------------------------------------------------------
# first, include gdt project to do some general configuration stuff
# (build modes, glut, optix, etc)
# ------------------------------------------------------------------
set(gdt_dir ${PROJECT_SOURCE_DIR}/common/gdt/)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${gdt_dir}/cmake/")
include(${gdt_dir}/cmake/configure_build_type.cmake)
include(${gdt_dir}/cmake/configure_optix.cmake)

mark_as_advanced(CUDA_SDK_ROOT_DIR)

# ------------------------------------------------------------------
# import gdt submodule
# ------------------------------------------------------------------
include_directories(${gdt_dir})
add_subdirectory(${gdt_dir} EXCLUDE_FROM_ALL)

# ------------------------------------------------------------------
# build glfw
# ------------------------------------------------------------------
set(OpenGL_GL_PREFERENCE LEGACY)
if (WIN32)
  set(glfw_dir ${PROJECT_SOURCE_DIR}/common/3rdParty/glfw/)
  include_directories(${glfw_dir}/include)
  add_subdirectory(${glfw_dir} EXCLUDE_FROM_ALL)
else()
  find_package(glfw3 REQUIRED)
endif()

include_directories(common)
add_subdirectory(common/glfWindow EXCLUDE_FROM_ALL)


# ------------------------------------------------------------------
# and final build rules for the project
# ------------------------------------------------------------------

set(optix_LIBRARY "")

include_directories(${OptiX_INCLUDE})

cuda_compile_and_embed(embedded_ptx_code devicePrograms.cu)

add_executable(${PROJECT_NAME}
  ${embedded_ptx_code}
  optix7.h
  CUDABuffer.h
  SampleRenderer.h
  SampleRenderer.cpp  
  main.cpp
  )
target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_11)
set_target_properties( ${PROJECT_NAME}  PROPERTIES CUDA_SEPARABLE_COMPILATION ON)

#attach devil library https://openil.sourceforge.net/download.php
include_directories(common/3rdParty/DevIL/include)
# copy libs
file(COPY ${PROJECT_SOURCE_DIR}/common/3rdParty/DevIL/lib/x64/Release/ DESTINATION ${CMAKE_SOURCE_DIR}/build/Debug)
file(COPY ${PROJECT_SOURCE_DIR}/common/3rdParty/DevIL/lib/x64/Release/ DESTINATION ${CMAKE_SOURCE_DIR}/build/Release)

target_link_libraries(${PROJECT_NAME}
  gdt
  # optix dependencies, for rendering
  ${optix_LIBRARY}
  ${CUDA_LIBRARIES}
  ${CUDA_CUDA_LIBRARY}
  # glfw and opengl, for display
  glfWindow
  glfw
  ${OPENGL_gl_LIBRARY}
  optimized ${CMAKE_BINARY_DIR}/Release/DevIL.lib
  optimized ${CMAKE_BINARY_DIR}/Release/ILU.lib
  optimized ${CMAKE_BINARY_DIR}/Release/ILUT.lib

 debug ${CMAKE_BINARY_DIR}/Debug/DevIL.lib
 debug ${CMAKE_BINARY_DIR}/Debug/ILU.lib
 debug ${CMAKE_BINARY_DIR}/Debug/ILUT.lib
  )

# set woeking directory and set startup project for MSVC
set_target_properties( ${PROJECT_NAME} PROPERTIES  VS_DEBUGGER_WORKING_DIRECTORY "$<TARGET_FILE_DIR:${PROJECT_NAME}>")
set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT ${PROJECT_NAME})
