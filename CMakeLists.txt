cmake_minimum_required(VERSION 3.15...3.30)
if(NOT SKBUILD_PROJECT_NAME)
  set(SKBUILD_PROJECT_NAME "pymol")
endif()

project(${SKBUILD_PROJECT_NAME} LANGUAGES C CXX)

include(CheckIncludeFileCXX)

find_package(Python REQUIRED COMPONENTS Interpreter Development.Module NumPy)

option(PYMOL_DEBUG "Enable debug-friendly compiler flags" OFF)
option(PYMOL_GLUT "Link against GLUT (legacy GUI)" OFF)
option(PYMOL_OSX_FRAMEWORKS "Use macOS frameworks for OpenGL/GLUT" ON)
option(PYMOL_LIBXML "Enable libxml2/COLLADA support" ON)
set(PYMOL_USE_OPENMP "default" CACHE STRING "OpenMP mode: default, yes, no")
set_property(CACHE PYMOL_USE_OPENMP PROPERTY STRINGS default yes no)
set(PYMOL_USE_VTKM "no" CACHE STRING "VTK-m version to use (2.0, 2.1, 2.2, 2.3, or no)")
set_property(CACHE PYMOL_USE_VTKM PROPERTY STRINGS 2.0 2.1 2.2 2.3 no)
set(PYMOL_USE_MSGPACKC "guess" CACHE STRING "msgpack mode: c++11, c, guess, no")
set_property(CACHE PYMOL_USE_MSGPACKC PROPERTY STRINGS c++11 c guess no)
option(PYMOL_TESTING "Build C-level tests" OFF)
option(PYMOL_OPENVR "Enable OpenVR support" OFF)
option(PYMOL_VMD_PLUGINS "Enable VMD molfile plugins" ON)
set(PYMOL_PREFIX_PATH "$ENV{PREFIX_PATH}" CACHE STRING "Additional prefix paths searched for headers/libraries")

if(DEFINED ENV{DEBUG} AND NOT "$ENV{DEBUG}" STREQUAL "")
  set(PYMOL_DEBUG ON)
endif()

set(_valid_openmp_values default yes no)
list(FIND _valid_openmp_values "${PYMOL_USE_OPENMP}" _openmp_mode_valid)
if(_openmp_mode_valid EQUAL -1)
  message(FATAL_ERROR "Invalid PYMOL_USE_OPENMP='${PYMOL_USE_OPENMP}'. Expected one of: default, yes, no")
endif()

set(_valid_msgpack_values "c++11" c guess no)
list(FIND _valid_msgpack_values "${PYMOL_USE_MSGPACKC}" _msgpack_mode_valid)
if(_msgpack_mode_valid EQUAL -1)
  message(FATAL_ERROR "Invalid PYMOL_USE_MSGPACKC='${PYMOL_USE_MSGPACKC}'. Expected one of: c++11, c, guess, no")
endif()

set(_valid_vtkm_values 2.0 2.1 2.2 2.3 no)
list(FIND _valid_vtkm_values "${PYMOL_USE_VTKM}" _vtkm_mode_valid)
if(_vtkm_mode_valid EQUAL -1)
  message(FATAL_ERROR "Invalid PYMOL_USE_VTKM='${PYMOL_USE_VTKM}'. Expected one of: 2.0, 2.1, 2.2, 2.3, no")
endif()

function(_append_existing_dirs out_var prefix)
  foreach(_suffix IN LISTS ARGN)
    set(_candidate "${prefix}/${_suffix}")
    if(IS_DIRECTORY "${_candidate}")
      list(APPEND ${out_var} "${_candidate}")
    endif()
  endforeach()
  set(${out_var} "${${out_var}}" PARENT_SCOPE)
endfunction()

function(_gather_sources out_var)
  set(_sources "")
  foreach(_dir IN LISTS ARGN)
    file(GLOB _dir_sources
      "${CMAKE_CURRENT_SOURCE_DIR}/${_dir}/*.c"
      "${CMAKE_CURRENT_SOURCE_DIR}/${_dir}/*.cpp"
    )
    list(APPEND _sources ${_dir_sources})
  endforeach()
  list(SORT _sources)
  set(${out_var} "${_sources}" PARENT_SCOPE)
endfunction()

set(_prefix_paths "")
if(WIN32 AND DEFINED ENV{VCPKG_ROOT})
  if(IS_DIRECTORY "$ENV{VCPKG_ROOT}/installed/x64-windows")
    list(APPEND _prefix_paths "$ENV{VCPKG_ROOT}/installed/x64-windows")
  endif()
endif()

if(PYMOL_PREFIX_PATH)
  set(_prefix_path_list "${PYMOL_PREFIX_PATH}")
  if(NOT WIN32)
    string(REPLACE ":" ";" _prefix_path_list "${_prefix_path_list}")
  endif()
  list(APPEND _prefix_paths ${_prefix_path_list})
endif()

if(WIN32 AND DEFINED ENV{CONDA_PREFIX})
  list(APPEND _prefix_paths "$ENV{CONDA_PREFIX}/Library")
endif()

list(APPEND _prefix_paths ${CMAKE_PREFIX_PATH})
list(REMOVE_DUPLICATES _prefix_paths)

set(PYMOL_GENERATED_DIR "${CMAKE_BINARY_DIR}/generated")
file(GLOB _shader_inputs "${CMAKE_CURRENT_SOURCE_DIR}/data/shaders/*")
add_custom_command(
  OUTPUT
    "${PYMOL_GENERATED_DIR}/ShaderText.h"
    "${PYMOL_GENERATED_DIR}/ShaderText.cpp"
    "${PYMOL_GENERATED_DIR}/PyMOLBuildInfo.h"
  COMMAND "${CMAKE_COMMAND}" -E make_directory "${PYMOL_GENERATED_DIR}"
  COMMAND "${Python_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/create_shadertext.py"
    "${CMAKE_CURRENT_SOURCE_DIR}/data/shaders"
    "${PYMOL_GENERATED_DIR}"
    "${PYMOL_GENERATED_DIR}/ShaderText.h"
    "${PYMOL_GENERATED_DIR}/ShaderText.cpp"
  DEPENDS
    "${CMAKE_CURRENT_SOURCE_DIR}/create_shadertext.py"
    ${_shader_inputs}
  WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
  COMMENT "Generating ShaderText and PyMOLBuildInfo headers"
  VERBATIM
)

set(pymol_source_dirs
  "ov/src"
  "layer0"
  "layer1"
  "layer2"
  "layer3"
  "layer4"
  "layer5"
  "layerGraphics"
  "layerGraphics/gl"
)

if(PYMOL_VMD_PLUGINS)
  list(APPEND pymol_source_dirs "contrib/uiuc/plugins/molfile_plugin/src")
endif()

if(PYMOL_TESTING)
  list(APPEND pymol_source_dirs "layerCTest")
endif()

if(PYMOL_OPENVR)
  list(APPEND pymol_source_dirs "contrib/vr")
endif()

set(_pymol_use_msgpackc "${PYMOL_USE_MSGPACKC}")

set(pymol_include_dirs
  "${CMAKE_CURRENT_SOURCE_DIR}/include"
  "${CMAKE_CURRENT_SOURCE_DIR}/contrib/pocketfft"
  # Source tree include roots (headers are included as e.g. "Base.h", "Setting.h", "ov_types.h")
  "${CMAKE_CURRENT_SOURCE_DIR}/ov/src"
  "${CMAKE_CURRENT_SOURCE_DIR}/layer0"
  "${CMAKE_CURRENT_SOURCE_DIR}/layer1"
  "${CMAKE_CURRENT_SOURCE_DIR}/layer2"
  "${CMAKE_CURRENT_SOURCE_DIR}/layer3"
  "${CMAKE_CURRENT_SOURCE_DIR}/layer4"
  "${CMAKE_CURRENT_SOURCE_DIR}/layer5"
  "${CMAKE_CURRENT_SOURCE_DIR}/layerGraphics"
  "${CMAKE_CURRENT_SOURCE_DIR}/layerGraphics/gl"
  "${CMAKE_CURRENT_SOURCE_DIR}/contrib/uiuc/plugins/molfile_plugin/src"
  "${PYMOL_GENERATED_DIR}"
  ${Python_INCLUDE_DIRS}
  ${Python_NumPy_INCLUDE_DIRS}
)
if(PYMOL_VMD_PLUGINS)
  list(APPEND pymol_include_dirs "${CMAKE_CURRENT_SOURCE_DIR}/contrib/uiuc/plugins/include")
endif()

if(WIN32 AND DEFINED ENV{VCPKG_ROOT})
  set(_vcpkg_libxml2_include "$ENV{VCPKG_ROOT}/installed/x64-windows/include/libxml2")
  if(IS_DIRECTORY "${_vcpkg_libxml2_include}")
    list(APPEND pymol_include_dirs "${_vcpkg_libxml2_include}")
  endif()
endif()

set(pymol_library_dirs "")
foreach(_prefix IN LISTS _prefix_paths)
  _append_existing_dirs(
    pymol_include_dirs "${_prefix}"
    "include" "include/freetype2" "include/libxml2" "include/openvr"
  )
  _append_existing_dirs(pymol_library_dirs "${_prefix}" "lib64" "lib")
endforeach()

set(CMAKE_REQUIRED_INCLUDES ${pymol_include_dirs})
if(_pymol_use_msgpackc STREQUAL "guess")
  check_include_file_cxx("msgpack/version_master.h" _pymol_has_msgpack_h)
  check_include_file_cxx("msgpack/version_master.hpp" _pymol_has_msgpack_hpp)
  if(_pymol_has_msgpack_h OR _pymol_has_msgpack_hpp)
    set(_pymol_use_msgpackc "c++11")
  else()
    set(_pymol_use_msgpackc "no")
  endif()
endif()
unset(CMAKE_REQUIRED_INCLUDES)

if(NOT _pymol_use_msgpackc STREQUAL "no")
  list(APPEND pymol_source_dirs "contrib/mmtf-c")
endif()

_gather_sources(pymol_sources ${pymol_source_dirs})
list(APPEND pymol_sources "${PYMOL_GENERATED_DIR}/ShaderText.cpp")

set(pymol_compile_definitions
  "_PYMOL_LIBPNG"
  "_PYMOL_FREETYPE"
  "_PYMOL_NUMPY"
)

if(PYMOL_LIBXML)
  list(APPEND pymol_compile_definitions "_HAVE_LIBXML")
endif()

if(NOT PYMOL_GLUT)
  list(APPEND pymol_compile_definitions "_PYMOL_NO_MAIN")
endif()

if(PYMOL_TESTING)
  list(APPEND pymol_compile_definitions "_PYMOL_CTEST")
endif()

if(PYMOL_OPENVR)
  list(APPEND pymol_compile_definitions "_PYMOL_OPENVR")
endif()

if(PYMOL_VMD_PLUGINS)
  list(APPEND pymol_compile_definitions "_PYMOL_VMD_PLUGINS")
endif()

if(_pymol_use_msgpackc STREQUAL "no")
  list(APPEND pymol_compile_definitions "_PYMOL_NO_MSGPACKC")
elseif(_pymol_use_msgpackc STREQUAL "c++11")
  list(APPEND pymol_compile_definitions "MMTF_MSGPACK_USE_CPP11" "MSGPACK_NO_BOOST")
endif()

if(WIN32)
  list(APPEND pymol_compile_definitions "WIN32")
elseif(APPLE)
  list(APPEND pymol_compile_definitions "PYMOL_CURVE_VALIDATE")
  if(PYMOL_OSX_FRAMEWORKS)
    list(APPEND pymol_compile_definitions "_PYMOL_OSX")
  endif()
endif()

if(PYMOL_DEBUG AND NOT WIN32)
  list(APPEND pymol_compile_definitions "_GLIBCXX_ASSERTIONS")
endif()

set(_pymol_use_openmp "${PYMOL_USE_OPENMP}")
if(_pymol_use_openmp STREQUAL "default")
  if(APPLE)
    set(_pymol_use_openmp "no")
  else()
    set(_pymol_use_openmp "yes")
  endif()
endif()

set(pymol_link_libraries "")
set(pymol_link_options "")
if(WIN32)
  set(_png_library_name "libpng")
  if(DEFINED ENV{VCPKG_ROOT} AND IS_DIRECTORY "$ENV{VCPKG_ROOT}/installed/x64-windows")
    set(_png_library_name "libpng16")
  endif()

  list(APPEND pymol_link_libraries
    "Advapi32"
    "Ws2_32"
    "glew32"
    "freetype"
    "${_png_library_name}"
    "opengl32"
  )

  if(PYMOL_GLUT)
    list(APPEND pymol_link_libraries "freeglut")
  endif()

  if(PYMOL_LIBXML)
    list(APPEND pymol_link_libraries "libxml2")
  endif()
else()
  list(APPEND pymol_link_libraries "png" "freetype")

  if(PYMOL_LIBXML)
    list(APPEND pymol_link_libraries "xml2")
  endif()

  if(APPLE)
    list(APPEND pymol_link_libraries "GLEW")
    if(PYMOL_OSX_FRAMEWORKS)
      list(APPEND pymol_link_options "-framework" "OpenGL")
      if(PYMOL_GLUT)
        list(APPEND pymol_link_options "-framework" "GLUT")
      endif()
    else()
      list(APPEND pymol_link_libraries "GL")
      if(PYMOL_GLUT)
        list(APPEND pymol_link_libraries "glut")
      endif()
    endif()
  else()
    list(APPEND pymol_link_libraries "GL" "GLEW")
    if(PYMOL_GLUT)
      list(APPEND pymol_link_libraries "glut")
    endif()
  endif()
endif()

if(_pymol_use_msgpackc STREQUAL "c")
  list(APPEND pymol_link_libraries "msgpackc")
endif()

if(PYMOL_VMD_PLUGINS)
  list(APPEND pymol_link_libraries "netcdf")
endif()

if(PYMOL_OPENVR)
  list(APPEND pymol_link_libraries "openvr_api")
endif()

if(NOT PYMOL_USE_VTKM STREQUAL "no")
  set(_vtkm_include_dir "")
  foreach(_prefix IN LISTS _prefix_paths)
    set(_vtkm_candidate "${_prefix}/include/vtkm-${PYMOL_USE_VTKM}")
    if(IS_DIRECTORY "${_vtkm_candidate}")
      set(_vtkm_include_dir "${_vtkm_candidate}")
      break()
    endif()
  endforeach()

  if(NOT _vtkm_include_dir)
    message(FATAL_ERROR "VTK-m headers not found for version ${PYMOL_USE_VTKM}. Set PREFIX_PATH or PYMOL_PREFIX_PATH.")
  endif()

  list(APPEND pymol_compile_definitions "_PYMOL_VTKM")
  list(APPEND pymol_include_dirs
    "${_vtkm_include_dir}"
    "${_vtkm_include_dir}/vtkm/thirdparty/diy/vtkmdiy/include"
    "${_vtkm_include_dir}/vtkm/thirdparty/lcl/vtkmlcl"
  )
  list(APPEND pymol_link_libraries
    "vtkm_cont-${PYMOL_USE_VTKM}"
    "vtkm_filter_contour-${PYMOL_USE_VTKM}"
    "vtkm_filter_core-${PYMOL_USE_VTKM}"
  )
endif()

set(pymol_compile_options "")
if(MSVC)
  list(APPEND pymol_compile_options "/MP" "/std:c++17")
  if(PYMOL_DEBUG)
    list(APPEND pymol_compile_options "/Z7")
    list(APPEND pymol_link_options "/DEBUG")
  endif()
else()
  list(APPEND pymol_compile_options
    "-Werror=return-type"
    "-Wunused-variable"
    "-Wno-switch"
    "-Wno-narrowing"
    "-Wno-char-subscripts"
  )
  if(PYMOL_DEBUG)
    list(APPEND pymol_compile_options "-Og" "-g")
  else()
    list(APPEND pymol_compile_options "-O3")
  endif()

  if(APPLE)
    list(APPEND pymol_compile_options "-fno-strict-aliasing")
  endif()
endif()

list(REMOVE_DUPLICATES pymol_include_dirs)
list(REMOVE_DUPLICATES pymol_library_dirs)
list(REMOVE_DUPLICATES pymol_link_libraries)

Python_add_library(pymol_cmd MODULE ${pymol_sources} WITH_SOABI)
set_target_properties(
  pymol_cmd
  PROPERTIES
    OUTPUT_NAME "_cmd"
    PREFIX ""
)
target_compile_features(pymol_cmd PRIVATE cxx_std_17)
target_include_directories(pymol_cmd PRIVATE ${pymol_include_dirs})
target_link_directories(pymol_cmd PRIVATE ${pymol_library_dirs})
target_compile_definitions(pymol_cmd PRIVATE ${pymol_compile_definitions})
target_compile_options(pymol_cmd PRIVATE ${pymol_compile_options})
if(pymol_link_options)
  target_link_options(pymol_cmd PRIVATE ${pymol_link_options})
endif()
target_link_libraries(pymol_cmd PRIVATE ${pymol_link_libraries})

if(_pymol_use_openmp STREQUAL "yes")
  find_package(OpenMP)
  if(OpenMP_CXX_FOUND)
    target_compile_definitions(pymol_cmd PRIVATE "PYMOL_OPENMP")
    target_link_libraries(pymol_cmd PRIVATE OpenMP::OpenMP_CXX)
  else()
    message(WARNING "OpenMP requested but not found. Continuing without OpenMP support.")
  endif()
endif()

file(GLOB champ_sources
  "${CMAKE_CURRENT_SOURCE_DIR}/contrib/champ/*.c"
  "${CMAKE_CURRENT_SOURCE_DIR}/contrib/champ/*.cpp"
)
list(SORT champ_sources)

Python_add_library(champ MODULE ${champ_sources} WITH_SOABI)
set_target_properties(
  champ
  PROPERTIES
    OUTPUT_NAME "_champ"
    PREFIX ""
)
target_compile_features(champ PRIVATE cxx_std_17)
target_include_directories(
  champ PRIVATE
  "${CMAKE_CURRENT_SOURCE_DIR}/contrib/champ"
  ${Python_INCLUDE_DIRS}
)
target_link_directories(champ PRIVATE ${pymol_library_dirs})

install(TARGETS pymol_cmd LIBRARY DESTINATION "pymol" RUNTIME DESTINATION "pymol")
install(TARGETS champ LIBRARY DESTINATION "chempy/champ" RUNTIME DESTINATION "chempy/champ")

install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/data" DESTINATION "pymol/pymol_path")
install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/test" DESTINATION "pymol/pymol_path")
install(DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/examples" DESTINATION "pymol/pymol_path")
install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE" DESTINATION "pymol/pymol_path")

if(PYMOL_OPENVR)
  install(
    FILES "${CMAKE_CURRENT_SOURCE_DIR}/contrib/vr/README.md"
    DESTINATION "pymol/pymol_path"
    RENAME "README-VR.txt"
  )
endif()
