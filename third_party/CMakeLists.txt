# Copyright (c) the JPEG XL Project
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

if((SANITIZER STREQUAL "asan") OR (SANITIZER STREQUAL "msan"))
  set(BUILD_TESTING OFF)
endif()

# Highway
set(HWY_SYSTEM_GTEST ON CACHE INTERNAL "")
set(HWY_FORCE_STATIC_LIBS ON CACHE INTERNAL "")
set(HWY_ENABLE_CONTRIB OFF CACHE INTERNAL "")
set(HWY_ENABLE_EXAMPLES OFF CACHE INTERNAL "")
set(HWY_ENABLE_LIST_TARGETS ON CACHE INTERNAL "")
set(HWY_ENABLE_TESTS OFF CACHE INTERNAL "")
if((SANITIZER STREQUAL "asan") OR (SANITIZER STREQUAL "msan"))
  set(HWY_ENABLE_INSTALL OFF CACHE INTERNAL "")
endif()
if (EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/highway/CMakeLists.txt" AND
    NOT JPEGXL_FORCE_SYSTEM_HWY)
  add_subdirectory(highway)
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/highway/LICENSE"
                 ${PROJECT_BINARY_DIR}/LICENSE.highway COPYONLY)
else()
  find_package(HWY 1.0.7)
  if (NOT HWY_FOUND)
    message(FATAL_ERROR
        "Highway library (hwy) not found. Install libhwy-dev or download it "
        "to third_party/highway from https://github.com/google/highway . "
        "Highway is required to build JPEG XL. You can run "
        "${PROJECT_SOURCE_DIR}/deps.sh to download this dependency.")
  endif()
  if(JPEGXL_DEP_LICENSE_DIR)
    configure_file("${JPEGXL_DEP_LICENSE_DIR}/libhwy-dev/copyright"
                   ${PROJECT_BINARY_DIR}/LICENSE.highway COPYONLY)
  endif()  # JPEGXL_DEP_LICENSE_DIR
endif()

# brotli
if (NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/brotli/c/include/brotli/decode.h" OR
    JPEGXL_FORCE_SYSTEM_BROTLI)
  find_package(Brotli)
  if (NOT Brotli_FOUND)
    message(FATAL_ERROR
        "Brotli not found, install brotli-dev or download brotli source code to"
        " third_party/brotli from https://github.com/google/brotli. You can use"
        " ${PROJECT_SOURCE_DIR}/deps.sh to download this dependency.")
  endif ()
  if(JPEGXL_DEP_LICENSE_DIR)
    configure_file("${JPEGXL_DEP_LICENSE_DIR}/libbrotli-dev/copyright"
                   ${PROJECT_BINARY_DIR}/LICENSE.brotli COPYONLY)
  endif()  # JPEGXL_DEP_LICENSE_DIR
else()
  # Compile brotli from sources.
  set(BROTLI_DISABLE_TESTS ON CACHE STRING "Disable Brotli tests")
  set(BROTLI_BUILD_TOOLS OFF CACHE STRING "")
  # Override default "no-install" policy.
  if((NOT SANITIZER STREQUAL "asan") AND (NOT SANITIZER STREQUAL "msan"))
    set(BROTLI_BUNDLED_MODE OFF CACHE INTERNAL "")
  endif()
  add_subdirectory(brotli)
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/brotli/LICENSE"
                 ${PROJECT_BINARY_DIR}/LICENSE.brotli COPYONLY)
  if(APPLE)
    if(NOT DEFINED CMAKE_MACOSX_RPATH)
      # Use @rpath in install_name when CMAKE_MACOSX_RPATH is not set.
      set_property(TARGET brotlienc PROPERTY MACOSX_RPATH TRUE)
      set_property(TARGET brotlidec PROPERTY MACOSX_RPATH TRUE)
      set_property(TARGET brotlicommon PROPERTY MACOSX_RPATH TRUE)
    endif()
    if((NOT DEFINED CMAKE_MACOSX_RPATH) OR CMAKE_MACOSX_RPATH)
      # Set library search path when @rpath is used.
      if(NOT DEFINED CMAKE_INSTALL_RPATH)
        set_property(TARGET brotlienc PROPERTY INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
        set_property(TARGET brotlidec PROPERTY INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
        set_property(TARGET brotlicommon PROPERTY INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
      endif()
    else()
      # Set conventional install_name when @rpath is not used.
      if(NOT DEFINED CMAKE_INSTALL_NAME_DIR)
        set_property(TARGET brotlienc PROPERTY INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/lib")
        set_property(TARGET brotlidec PROPERTY INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/lib")
        set_property(TARGET brotlicommon PROPERTY INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/lib")
      endif()
    endif()
  endif()  # APPLE
endif()

# *cms
if (JPEGXL_ENABLE_SKCMS)
  if( NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/skcms/skcms.h" )
    message(FATAL_ERROR "Please provide skcms or "
            "run ${PROJECT_SOURCE_DIR}/deps.sh to fetch the build dependencies.")
  endif()
  include(skcms.cmake)
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/skcms/LICENSE"
                 ${PROJECT_BINARY_DIR}/LICENSE.skcms COPYONLY)
endif ()
if (JPEGXL_ENABLE_VIEWERS OR NOT JPEGXL_ENABLE_SKCMS OR JPEGXL_ENABLE_PLUGINS)
  if( NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/lcms/.git" OR JPEGXL_FORCE_SYSTEM_LCMS2 )
    find_package(LCMS2 2.12)
    if ( NOT LCMS2_FOUND )
      message(FATAL_ERROR "Please provide lcms2 or run git submodule update --init")
    endif ()
  else()
    include(lcms2.cmake)
    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/lcms/LICENSE"
                   ${PROJECT_BINARY_DIR}/LICENSE.lcms COPYONLY)
  endif()
endif()

# libpng
if (JPEGXL_BUNDLE_LIBPNG AND EMSCRIPTEN)
  if (NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libpng/CMakeLists.txt")
    message(FATAL_ERROR "Please provide libpng or "
            "run ${PROJECT_SOURCE_DIR}/deps.sh to fetch the build dependencies.")
  endif()
  file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/libpng/scripts/pnglibconf.h.prebuilt" DESTINATION "${CMAKE_CURRENT_SOURCE_DIR}/libpng")
  file(RENAME "${CMAKE_CURRENT_SOURCE_DIR}/libpng/pnglibconf.h.prebuilt" "${CMAKE_CURRENT_SOURCE_DIR}/libpng/pnglibconf.h")
  set(ZLIB_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/zlib/")
  set(ZLIB_LIBRARIES "")
  set(PNG_FOUND YES PARENT_SCOPE)
  set(PNG_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/libpng/" PARENT_SCOPE)
  set(PNG_LIBRARIES "" PARENT_SCOPE)
elseif (JPEGXL_BUNDLE_LIBPNG)
  if (NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/libpng/CMakeLists.txt")
    message(FATAL_ERROR "Please provide libpng or "
            "run ${PROJECT_SOURCE_DIR}/deps.sh to fetch the build dependencies.")
  endif()
  file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/libpng/scripts/pnglibconf.h.prebuilt" DESTINATION "${CMAKE_CURRENT_SOURCE_DIR}/libpng")
  file(RENAME "${CMAKE_CURRENT_SOURCE_DIR}/libpng/pnglibconf.h.prebuilt" "${CMAKE_CURRENT_SOURCE_DIR}/libpng/pnglibconf.h")
  # Build only the static zlib so that the RC resource file (win32/zlib1.rc)
  # is not processed by the resource compiler, which fails with ClangCL on ARM64
  # because it can't parse modern Clang ARM64 system headers.
  set(_saved_BUILD_SHARED_LIBS ${BUILD_SHARED_LIBS})
  set(BUILD_SHARED_LIBS OFF)
  add_subdirectory(zlib)
  set(BUILD_SHARED_LIBS ${_saved_BUILD_SHARED_LIBS})
  # After add_subdirectory(zlib), the binary dir has the generated zconf.h.
  # Copy zlib.h there too so that ZLIB_INCLUDE_DIR can point to a single
  # directory that contains both headers, which FindZLIB expects.
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/zlib/zlib.h"
                 "${CMAKE_CURRENT_BINARY_DIR}/zlib/zlib.h" COPYONLY)
  set(PNG_STATIC ON CACHE BOOL "")
  set(PNG_TOOLS OFF CACHE BOOL "")
  set(ZLIB_BUILD_EXAMPLES OFF CACHE BOOL "")
  # Point ZLIB_INCLUDE_DIR at the binary dir, which now holds both zlib.h
  # (just copied) and the generated zconf.h, so libpng's genout.cmake step
  # finds both headers with a single -I flag.
  set(ZLIB_INCLUDE_DIR "${CMAKE_CURRENT_BINARY_DIR}/zlib" CACHE PATH "" FORCE)
  set(ZLIB_LIBRARY bing_bong_lib)
  add_library(ZLIB::ZLIB INTERFACE IMPORTED)
  # Provide zlib headers to ZLIB::ZLIB so libpng's pnglibconf.c compile step
  # can find <zlib.h>.  We need both the source tree (zlib.h) and the binary
  # tree (generated zconf.h).
  set_target_properties(ZLIB::ZLIB PROPERTIES
    INTERFACE_INCLUDE_DIRECTORIES
      "${CMAKE_CURRENT_BINARY_DIR}/zlib")
  set(PNG_TESTS OFF CACHE BOOL "")
  set(SKIP_INSTALL_ALL ON CACHE BOOL "")
  # Include both the source dir (zlib.h) and the binary dir (generated zconf.h)
  # so that libpng's genout.cmake pnglibconf.c compile step can find both headers.
  set(ZLIB_INCLUDE_DIRS "${CMAKE_CURRENT_BINARY_DIR}/zlib")
  set(ZLIB_LIBRARIES zlibstatic)
  # When cross-compiling to ARM64 on an x64 host, CMAKE_SYSTEM_PROCESSOR is AMD64
  # (the host), so libpng's architecture detection sets PNG_TARGET_ARCHITECTURE to
  # "amd64" and skips the NEON sources.  Force PNG_ARM_NEON=on so that the NEON
  # intrinsic files are compiled — the ClangCL ARM64 toolchain defines __ARM_NEON
  # and requires those symbols to be present at link time.
  # Also override PNG_TARGET_ARCHITECTURE so that the initial ARM/NEON branch is
  # actually taken (the architecture check happens before PNG_ARM_NEON is read).
  if(CMAKE_GENERATOR_PLATFORM STREQUAL "ARM64" OR CMAKE_VS_PLATFORM_NAME STREQUAL "ARM64")
    set(PNG_ARM_NEON "on" CACHE STRING "Enable ARM NEON optimizations" FORCE)
    set(PNG_TARGET_ARCHITECTURE "arm64" CACHE STRING
        "Override libpng arch detection for ARM64 cross-compile" FORCE)
  endif()
  add_subdirectory(libpng EXCLUDE_FROM_ALL)
  set(PNG_FOUND YES PARENT_SCOPE)
  set(PNG_INCLUDE_DIRS "${CMAKE_CURRENT_SOURCE_DIR}/libpng/" PARENT_SCOPE)
  target_link_libraries(png_static PUBLIC zlibstatic)
  set(PNG_LIBRARIES png_static PARENT_SCOPE)
  set_property(TARGET png_static PROPERTY POSITION_INDEPENDENT_CODE ON)
  set_property(TARGET zlibstatic PROPERTY POSITION_INDEPENDENT_CODE ON)
  if(JPEGXL_DEP_LICENSE_DIR)
    configure_file("${CMAKE_CURRENT_SOURCE_DIR}/libpng/LICENSE"
                   ${PROJECT_BINARY_DIR}/LICENSE.libpng COPYONLY)
  endif()
else()
  find_package(PNG)
  if(PNG_FOUND AND JPEGXL_DEP_LICENSE_DIR)
    configure_file("${JPEGXL_DEP_LICENSE_DIR}/zlib1g-dev/copyright"
                   ${PROJECT_BINARY_DIR}/LICENSE.zlib COPYONLY)
    configure_file("${JPEGXL_DEP_LICENSE_DIR}/libpng-dev/copyright"
                   ${PROJECT_BINARY_DIR}/LICENSE.libpng COPYONLY)
  endif()  # JPEGXL_DEP_LICENSE_DIR
endif()

# sjpeg
if (JPEGXL_ENABLE_SJPEG)
  if (NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/sjpeg/CMakeLists.txt")
    message(FATAL_ERROR "Please provide sjpeg or "
            "run ${PROJECT_SOURCE_DIR}/deps.sh to fetch the build dependencies.")
  endif()
  include(sjpeg.cmake)
  configure_file("${CMAKE_CURRENT_SOURCE_DIR}/sjpeg/COPYING"
                 ${PROJECT_BINARY_DIR}/LICENSE.sjpeg COPYONLY)
endif ()
