
# Copyright(c) 2017 Intel Corporation

# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files(the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and / or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:

# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.

# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR
# OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE,
# ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
# OTHER DEALINGS IN THE SOFTWARE.


cmake_minimum_required(VERSION 3.5)
project(igfx_gmmumd)
if(NOT DEFINED BS_USE_OSDM_BUILD_SYSTEM)
    if(DEFINED ENV{BS_USE_OSDM_BUILD_SYSTEM})
        set(BS_USE_OSDM_BUILD_SYSTEM "$ENV{BS_USE_OSDM_BUILD_SYSTEM}")
    else()
        set(BS_USE_OSDM_BUILD_SYSTEM FALSE)
    endif()
endif()

# begin -- label bldsys file prologue
# WARNING: The "project" statement triggers reading of CMAKE_TOOLCHAIN_FILE
#   and so must precede the inclusion below of bs_init.cmake .
function(bs_find_build_system gfx_dev_dir build_sys_dir build_sys_inc)
    # If we are not building as a standalone project
    if(DEFINED GFX_DEVELOPMENT_DIR)
        set(_bs_gfx_development_dir "${GFX_DEVELOPMENT_DIR}")
    elseif(DEFINED ENV{GFX_DEVELOPMENT_DIR})
        set(_bs_gfx_development_dir "$ENV{GFX_DEVELOPMENT_DIR}")
    else()
        get_filename_component(_bs_cur_cmake_dir "${CMAKE_CURRENT_LIST_FILE}" PATH)
        get_filename_component(_bs_parent_dir "${_bs_cur_cmake_dir}" DIRECTORY)
        set(_bs_gfx_dir_found false)
        while(NOT _bs_gfx_dir_found)
            set(_bs_bldsys_dir "${_bs_parent_dir}/Tools/bldsys")
            if(EXISTS ${_bs_bldsys_dir})
                set(_bs_gfx_development_dir ${_bs_parent_dir})
                set(_bs_gfx_dir_found true)
                break()
            endif()
            get_filename_component(_bs_parent_dir "${_bs_parent_dir}" DIRECTORY)
            if(${_bs_parent_dir} STREQUAL "/")
                break()
            endif()
        endwhile(NOT _bs_gfx_dir_found)
        if (NOT _bs_gfx_development_dir)
            message(FATAL_ERROR "GFX_DEVELOPMENT_DIR not found (${_bs_gfx_development_dir}) - exiting!")
            exit(1)
        endif()
    endif()
    set(${gfx_dev_dir} "${_bs_gfx_development_dir}" PARENT_SCOPE)
    set(${build_sys_dir} "${_bs_gfx_development_dir}/Tools/bldsys" PARENT_SCOPE)
    set(${build_sys_inc} "${_bs_gfx_development_dir}/Tools/bldsys/include" PARENT_SCOPE)
endfunction(bs_find_build_system)

bs_find_build_system(GFX_DEVELOPMENT_DIR BUILD_SYS_DIR BUILD_SYS_INC)

include(${BUILD_SYS_DIR}/bs_init.cmake)
include(${BUILD_SYS_INC}/bs_dir_names.cmake)
# file prologue done


set(CMAKE_DISABLE_SOURCE_CHANGES ON)
set(CMAKE_DISABLE_IN_SOURCE_BUILD ON)

set (TARGET_LIST
        igfxgmmumd
	gmm_umd
	igfx_gmmumd_excite
	#igfx_gmmumd_ocl
	)

################################################################################
# Define GMM_DLL Target
################################################################################
set (GMM_LIB_DLL_NAME igfx_gmmumd_dll)

macro(GmmLibSetTargetConfig libTarget)
	if (TARGET ${libTarget})
			set_property(TARGET ${libTarget} APPEND PROPERTY COMPILE_DEFINITIONS
				$<$<CONFIG:Release>: _RELEASE>
				$<$<CONFIG:ReleaseInternal>: _RELEASE_INTERNAL>
				$<$<CONFIG:Debug>: _DEBUG>
			)
		endif()

endmacro()

set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /W1 /WX-")
set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} /W1 /WX-")

# GMMLIBsetLinkerFlags: apply linker flags for given configuration
# linkerFlags: linker specific options
# linkerTarget: optional parameter - apply linker flags for specfied target
macro (GMMLIBsetLinkerFlags linkerFlags linkerTarget)
    foreach (opt ${linkerFlags})
        if ("${linkerTarget}" STREQUAL "")
            set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${opt}")
            set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} ${opt}")
        else()
            set (CMAKE_SHARED_LINKER_FLAGS_${linkerTarget} "${CMAKE_SHARED_LINKER_FLAGS_${linkerTarget}} ${opt}")
            set (CMAKE_EXE_LINKER_FLAGS_${linkerTarget} "${CMAKE_SHARED_LINKER_FLAGS_${linkerTarget}} ${opt}")
        endif()
    endforeach()
endmacro()

set( CMAKE_CONFIGURATION_TYPES
    "Debug"
    "Release"
    "ReleaseInternal")

set( CMAKE_CONFIGURATION_TYPES "${CMAKE_CONFIGURATION_TYPES}" CACHE STRING
    "Reset the configurations to what we need"
    FORCE)


set(GMMLIB_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR})

if ("${ARCH}" MATCHES "64")
    set (GMMLIB_ARCH "64")
else()
    set (GMMLIB_ARCH "32")
endif()

if ("${BUILD_TYPE}" STREQUAL "release")
    set(CMAKE_BUILD_TYPE "Release")
elseif ("${BUILD_TYPE}" STREQUAL "release-internal")
    set(CMAKE_BUILD_TYPE "ReleaseInternal")
elseif ("${BUILD_TYPE}" STREQUAL "debug")
    set(CMAKE_BUILD_TYPE "Debug")
endif()

if("${GMMLIB_MARCH}" STREQUAL "")
    set(GMMLIB_MARCH "corei7")
endif()

MESSAGE("platform: ${CMAKE_HOST_SYSTEM_NAME}")
MESSAGE("source_dir: ${BS_DIR_GMMLIB}")
MESSAGE("arch: ${GMMLIB_ARCH}")
MESSAGE("build type: ${CMAKE_BUILD_TYPE}")
MESSAGE("SourceConfiguration:")
MESSAGE("CommonDir: ${BS_DIR_COMMON}")
MESSAGE("IncDir: ${BS_DIR_INC}")

# If '-DGMM_DYNAMIC_MOCS_TABLE=TRUE' (default is FALSE) passed to cmake
# configure command gmmlib will generate MOCS table dynamically depending on
# usage requests (Gen9+). In this case on Linux user responisbility is to
# make sure that generated MOCS table is programmed on KMD level.
if (GMM_DYNAMIC_MOCS_TABLE)
    MESSAGE("MOCS table: dynamic")
    add_definitions(-DGMM_DYNAMIC_MOCS_TABLE)
else()
    MESSAGE("MOCS table: static")
endif()

if(DEFINED UFO_DRIVER_OPTIMIZATION_LEVEL)
    if(${UFO_DRIVER_OPTIMIZATION_LEVEL} GREATER 0)
        add_definitions(-DGMM_GFX_GEN=${GFXGEN})
    endif()
endif()

set(HEADERS_
	${BS_DIR_GMMLIB}/CachePolicy/GmmCachePolicyConditionals.h
	${BS_DIR_GMMLIB}/CachePolicy/GmmCachePolicyResourceUsageDefinitions.h
	${BS_DIR_GMMLIB}/CachePolicy/GmmCachePolicyUndefineConditionals.h
	${BS_DIR_GMMLIB}/CachePolicy/GmmGen10CachePolicy.h
	${BS_DIR_GMMLIB}/CachePolicy/GmmGen8CachePolicy.h
	${BS_DIR_GMMLIB}/CachePolicy/GmmGen9CachePolicy.h
	${BS_DIR_GMMLIB}/inc/External/Common/CachePolicy/GmmCachePolicyGen10.h
	${BS_DIR_GMMLIB}/inc/External/Common/CachePolicy/GmmCachePolicyGen8.h
	${BS_DIR_GMMLIB}/inc/External/Common/CachePolicy/GmmCachePolicyGen9.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmCachePolicy.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmCachePolicyCommon.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmCachePolicyExt.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmCommonExt.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmConst.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmDebug.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmFormatTable.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmGttExt.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmInfo.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmInfoExt.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmInternal.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmMemAllocator.hpp

	${BS_DIR_GMMLIB}/inc/External/Common/GmmPlatformExt.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmProto.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmResourceFlags.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmResourceInfo.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmResourceInfoCommon.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmResourceInfoExt.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmTextureExt.h
	${BS_DIR_GMMLIB}/inc/External/Common/GmmUtil.h
	${BS_DIR_GMMLIB}/inc/External/Linux/GmmResourceInfoLin.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/Platform/GmmGen10Platform.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/Platform/GmmGen8Platform.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/Platform/GmmGen9Platform.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/Texture/GmmGen10TextureCalc.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/Texture/GmmGen7TextureCalc.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/Texture/GmmGen8TextureCalc.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/Texture/GmmGen9TextureCalc.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/Texture/GmmTextureCalc.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/GmmCommonInt.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/GmmLibInc.h
	${BS_DIR_GMMLIB}/inc/Internal/Common/GmmTextureCalc.h
	${BS_DIR_GMMLIB}/inc/GmmLib.h
)

set(UMD_HEADERS
	${HEADERS_}
	${BS_DIR_GMMLIB}/inc/External/Common/GmmClientContext.h
        ${BS_DIR_GMMLIB}/inc/External/Common/GmmLibDll.h
	)


set(SOURCES_
  ${BS_DIR_COMMON}/AssertTracer/AssertTracer.cpp
  ${BS_DIR_GMMLIB}/CachePolicy/GmmCachePolicy.cpp
  ${BS_DIR_GMMLIB}/CachePolicy/GmmCachePolicyCommon.cpp
  ${BS_DIR_GMMLIB}/CachePolicy/GmmGen8CachePolicy.cpp
  ${BS_DIR_GMMLIB}/CachePolicy/GmmGen9CachePolicy.cpp
  ${BS_DIR_GMMLIB}/CachePolicy/GmmGen10CachePolicy.cpp
  ${BS_DIR_GMMLIB}/Platform/GmmGen8Platform.cpp
  ${BS_DIR_GMMLIB}/Platform/GmmGen9Platform.cpp
  ${BS_DIR_GMMLIB}/Platform/GmmGen10Platform.cpp
  ${BS_DIR_GMMLIB}/Platform/GmmPlatform.cpp
  ${BS_DIR_GMMLIB}/Resource/GmmResourceInfo.cpp
  ${BS_DIR_GMMLIB}/Resource/GmmResourceInfoCommon.cpp
  ${BS_DIR_GMMLIB}/Resource/GmmResourceInfoCommonEx.cpp
  ${BS_DIR_GMMLIB}/Resource/GmmRestrictions.cpp
  ${BS_DIR_GMMLIB}/Texture/GmmGen7Texture.cpp
  ${BS_DIR_GMMLIB}/Texture/GmmGen8Texture.cpp
  ${BS_DIR_GMMLIB}/Texture/GmmGen9Texture.cpp
  ${BS_DIR_GMMLIB}/Texture/GmmGen10Texture.cpp
  ${BS_DIR_GMMLIB}/Texture/GmmTexture.cpp
  ${BS_DIR_GMMLIB}/Texture/GmmTextureAlloc.cpp
  ${BS_DIR_GMMLIB}/Texture/GmmTextureSpecialCases.cpp
  ${BS_DIR_GMMLIB}/Texture/GmmTextureOffset.cpp
  ${BS_DIR_GMMLIB}/GlobalInfo/GmmInfo.cpp
  ${BS_DIR_GMMLIB}/Utility/CpuSwizzleBlt/CpuSwizzleBlt.c
  ${BS_DIR_GMMLIB}/Utility/GmmLibObject.cpp
  ${BS_DIR_GMMLIB}/Utility/GmmLog/GmmLog.cpp
  ${BS_DIR_GMMLIB}/Utility/GmmUtility.cpp
)

set(UMD_SOURCES
  ${SOURCES_}
  ${BS_DIR_GMMLIB}/GlobalInfo/GmmClientContext.cpp
  ${BS_DIR_GMMLIB}/GlobalInfo/GmmOldApi.cpp
   ${BS_DIR_GMMLIB}/GlobalInfo/GmmLibDllMain.cpp
  )

source_group("Source Files\\Cache Policy\\Client Files" FILES
			${BS_DIR_GMMLIB}/CachePolicy/GmmCachePolicyResourceUsageDefinitions.h
			${BS_DIR_GMMLIB}/CachePolicy/GmmGen10CachePolicy.h
			${BS_DIR_GMMLIB}/CachePolicy/GmmGen8CachePolicy.h
			${BS_DIR_GMMLIB}/CachePolicy/GmmGen9CachePolicy.h
			)

source_group("Source Files\\Cache Policy" ${BS_DIR_GMMLIB}/CachePolicy/*.cpp)
source_group("Source Files\\Global" ${BS_DIR_GMMLIB}/GlobalInfo/.*)
source_group("Source Files\\Platform" ${BS_DIR_GMMLIB}/Platform/.*)
source_group("Source Files\\Texture" ${BS_DIR_GMMLIB}/Texture/.*)
source_group("Source Files\\Utility" ${BS_DIR_GMMLIB}/Utility/.*)

source_group("Source Files\\Resource" FILES
			${BS_DIR_GMMLIB}/Resource/GmmResourceInfo.cpp
			${BS_DIR_GMMLIB}/Resource/GmmResourceInfoCommon.cpp
			${BS_DIR_GMMLIB}/Resource/GmmRestrictions.cpp)

source_group("Header Files\\External\\Common" FILES
			${BS_DIR_GMMLIB}/inc/External/Common/GmmCachePolicy.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmCachePolicyCommon.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmCachePolicyExt.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmCommonExt.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmConst.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmDebug.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmFormatTable.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmGttExt.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmInfo.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmInfoExt.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmInternal.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmMemAllocator.hpp
			${BS_DIR_GMMLIB}/inc/External/Common/GmmPageTableMgr.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmPlatformExt.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmResourceFlags.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmResourceInfo.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmResourceInfoCommon.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmResourceInfoExt.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmTextureExt.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmUtil.h
			${BS_DIR_GMMLIB}/inc/External/Common/GmmClientContext.h
			)

source_group("Header Files" FILES
			${BS_DIR_GMMLIB}/inc/GmmLib.h
			)

source_group("Header Files\\External\\Common\\Cache Policy" FILES
			${BS_DIR_GMMLIB}/inc/External/Common/CachePolicy/GmmCachePolicyGen10.h
			${BS_DIR_GMMLIB}/inc/External/Common/CachePolicy/GmmCachePolicyGen8.h
			${BS_DIR_GMMLIB}/inc/External/Common/CachePolicy/GmmCachePolicyGen9.h
			)

source_group("Header Files\\External\\Linux" FILES
			 ${BS_DIR_GMMLIB}/inc/External/Linux/GmmResourceInfoLin.h
			)

source_group("Header Files\\Internal\\Common" FILES
			${BS_DIR_GMMLIB}/inc/Internal/Common/GmmLibInc.h
			${BS_DIR_GMMLIB}/inc/Internal/Common/GmmProto.h
			)

source_group("Header Files\\Internal\\Common\\Platform" FILES
			${BS_DIR_GMMLIB}/inc/Internal/Common/Platform/GmmGen10Platform.h
			${BS_DIR_GMMLIB}/inc/Internal/Common/Platform/GmmGen8Platform.h
			${BS_DIR_GMMLIB}/inc/Internal/Common/Platform/GmmGen9Platform.h
			)

source_group("Header Files\\Internal\\Common\\Texture" FILES
			${BS_DIR_GMMLIB}/inc/Internal/Common/Texture/GmmGen10TextureCalc.h
			${BS_DIR_GMMLIB}/inc/Internal/Common/Texture/GmmGen7TextureCalc.h
			${BS_DIR_GMMLIB}/inc/Internal/Common/Texture/GmmGen8TextureCalc.h
			${BS_DIR_GMMLIB}/inc/Internal/Common/Texture/GmmGen9TextureCalc.h
			${BS_DIR_GMMLIB}/inc/Internal/Common/Texture/GmmTextureCalc.h
			)

include_directories(BEFORE ${BS_DIR_GMMLIB}/)

include_directories(BEFORE ${PROJECT_SOURCE_DIR})

bs_add_some_common_includes()

  include_directories(
  ${BS_DIR_GMMLIB}
  ${BS_DIR_GMMLIB}/Utility/GmmLog
  ${BS_DIR_GMMLIB}/inc
  ${BS_DIR_GMMLIB}/Utility
  ${BS_DIR_GMMLIB}/GlobalInfo
  ${BS_DIR_GMMLIB}/Texture
  ${BS_DIR_GMMLIB}/Resource
  ${BS_DIR_GMMLIB}/Platform
  ${BS_DIR_UTIL}
  ${BS_DIR_INC}
  ${BS_DIR_INC}/common
  ${BS_DIR_INC}/umKmInc
  ${BS_DIR_INSTALL}
  #${BS_DIR_ANDROID}/include
  )

set(headers
	${HEADERS_}
)

  set(SOURCES
    ${SOURCES_}
  )

#clear all default-cmake settings
SET (CMAKE_SHARED_LINKER_FLAGS_RELEASEINTERNAL "")
SET (CMAKE_SHARED_LINKER_FLAGS_RELEASE "")
SET (CMAKE_SHARED_LINKER_FLAGS_DEBUG "")
SET (CMAKE_SHARED_LINKER_FLAGS "")

SET (CMAKE_EXE_LINKER_FLAGS "")
SET (CMAKE_EXE_LINKER_FLAGS_DEBUG "")
SET (CMAKE_EXE_LINKER_FLAGS_RELEASEINTERNAL "")
SET (CMAKE_EXE_LINKER_FLAGS_RELEASE "")

SET (CMAKE_STATIC_LINKER_FLAGS "")
SET (CMAKE_LOCAL_LINKER_FLAGS "")

SET (CMAKE_CXX_STANDARD_LIBRARIES "")
SET (CMAKE_C_STANDARD_LIBRARIES "")

SET (CMAKE_CXX_FLAGS_RELEASEINTERNAL "")
SET (CMAKE_CXX_FLAGS_DEBUG  "")
SET (CMAKE_CXX_FLAGS_RELEASE "")

# set compiler options
if(MSVC)
	include(Windows.cmake)
else()
	include(Linux.cmake)
endif()

foreach(lib_target ${TARGET_LIST})
  add_library( ${lib_target} STATIC ${UMD_HEADERS} ${UMD_SOURCES})
endforeach()

foreach(lib_target ${TARGET_LIST})
	GmmLibSetTargetConfig(${lib_target})
endforeach()

###################################################################################
# create dll library
###################################################################################
add_library( ${GMM_LIB_DLL_NAME} SHARED igdgmm.rc ${UMD_SOURCES} ${UMD_HEADERS})

set_target_properties(${GMM_LIB_DLL_NAME} PROPERTIES OUTPUT_NAME "igdgmm${ARCH}")

if(MSVC)

bs_set_wdk(${GMM_LIB_DLL_NAME})
GmmLibSetTargetConfig( ${GMM_LIB_DLL_NAME} )

set_target_properties(${GMM_LIB_DLL_NAME} PROPERTIES VS_GLOBAL_DriverTargetPlatform Universal)
set_target_properties(${GMM_LIB_DLL_NAME} PROPERTIES VS_PLATFORM_TOOLSET WindowsApplicationForDrivers10.0)
windows_umd_props_universal(${GMM_LIB_DLL_NAME})

target_link_libraries( ${GMM_LIB_DLL_NAME}
            onecoreuap.lib
        )
endif()

###################################################################################
# End of DLL create library
###################################################################################

bs_set_defines()

if(MSVC)
	foreach(lib_target ${TARGET_LIST})
		bs_set_wdk(${lib_target})
	endforeach()
endif()

bs_set_force_exceptions()

bs_set_post_target()

foreach(lib_target ${TARGET_LIST})
  bs_set_extra_target_properties(${lib_target}
      _ATL_NO_WIN_SUPPORT
      SMALL_POOL_ALLOC
      __GMM
      __GFX_MACRO_C__
      UNUSED_ISTDLIB_MT
      __UMD
      GMM_UNIFY_DAF_API
	  )

	if(CMAKE_BUILD_TYPE STREQUAL "ReleaseInternal")
		bs_set_extra_target_properties(${lib_target} _RELEASE_INTERNAL)
	endif()

    target_include_directories(${lib_target} INTERFACE
        ${BS_DIR_GMMLIB}/inc
        ${BS_DIR_INC}
        ${BS_DIR_INC}/common)

endforeach()

###################################################################################
# Set common macros for DLL
###################################################################################
bs_set_extra_target_properties(${GMM_LIB_DLL_NAME}
      _ATL_NO_WIN_SUPPORT
      SMALL_POOL_ALLOC
      __GMM
      __GFX_MACRO_C__
      UNUSED_ISTDLIB_MT
	  __UMD
	  GMM_UNIFY_DAF_API
	  )

if(CMAKE_BUILD_TYPE STREQUAL "ReleaseInternal")
	bs_set_extra_target_properties(${GMM_LIB_DLL_NAME} _RELEASE_INTERNAL)
endif()

target_include_directories(${GMM_LIB_DLL_NAME} INTERFACE
        ${BS_DIR_GMMLIB}/inc
        ${BS_DIR_INC}
        ${BS_DIR_INC}/common)

###################################################################################
# End of Set macros DLL
###################################################################################

if ("${GMM_CLIENT_TYPE}" STREQUAL "OCL")
	bs_set_extra_target_properties(gmm_umd
	    GMM_OCL
            UNUSED_ISTDLIB_MT
        )
else()
    	bs_set_extra_target_properties(gmm_umd
	    GMM_OGL
	    UNUSED_ISTDLIB_MT
        )
endif()

#bs_set_extra_target_properties(igfx_gmmumd_ocl
#	GMM_OCL
#)

bs_set_extra_target_properties(igfxgmmumd
	ISTDLIB_UMD
	UNUSED_ISTDLIB_MT
	GMM_UNIFIED_LIB
)

bs_set_extra_target_properties(${GMM_LIB_DLL_NAME}
	ISTDLIB_UMD
	UNUSED_ISTDLIB_MT
	GMM_UNIFIED_LIB
	GMM_LIB_DLL
)

if("${ARCH}" MATCHES "64")
	bs_set_extra_target_properties(${GMM_LIB_DLL_NAME}
		_X64)
endif()

if(ARCH EQUAL 32)
	#bs_set_extra_target_properties(igfx_gmmumd_ocl UNUSED_ISTDLIB_MT)

	if((CMAKE_BUILD_TYPE STREQUAL "Release") OR (CMAKE_BUILD_TYPE STREQUAL "ReleaseInternal"))
		bs_set_extra_target_properties(gmm_umd ISTDLIB_UMD)
	endif()
endif()

add_subdirectory(ULT)
