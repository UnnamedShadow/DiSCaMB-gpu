#Specify the version being used aswell as the language
cmake_minimum_required(VERSION 3.15)

# CRITICAL: Clear init flags BEFORE project() to prevent default flag injection
set(CMAKE_CXX_FLAGS_INIT "")
set(CMAKE_C_FLAGS_INIT "")

# Disable problematic flags for GCC offload builds
string(REPLACE "-fcf-protection=full" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
string(REPLACE "-flto" "" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
string(REPLACE "-flto" "" CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}")

cmake_policy(SET CMP0091 NEW)

PROJECT(discamb LANGUAGES CXX C)

set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

SET_PROPERTY(GLOBAL PROPERTY USE_FOLDERS ON)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(DISCAMB_MODULES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

if(MSVC)
  add_definitions(/MP)
endif()

# Helper to strip flags unsupported by NVPTX/offload toolchain
function(strip_unsupported_flags out_var in_var)
  set(tmp "${${in_var}}")
  # remove -fcf-protection=full and any -flto occurrences
  string(REGEX REPLACE "-fcf-protection[^ ]*" "" tmp "${tmp}")
  string(REGEX REPLACE "-flto[^ ]*" "" tmp "${tmp}")
  string(REGEX REPLACE "-fstack-protector[^ ]*" "" tmp "${tmp}")
  string(REGEX REPLACE "-fstack-clash-protection" "" tmp "${tmp}")
  # trim repeated spaces
  string(REGEX REPLACE "[ 	]+" " " tmp "${tmp}")
  string(STRIP "${tmp}" tmp)
  set(${out_var} "${tmp}" PARENT_SCOPE)
endfunction()

# Strip unsupported flags from all build types
strip_unsupported_flags(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
strip_unsupported_flags(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
strip_unsupported_flags(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
strip_unsupported_flags(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
strip_unsupported_flags(CMAKE_CXX_FLAGS_RELWITHDEBINFO "${CMAKE_CXX_FLAGS_RELWITHDEBINFO}")
strip_unsupported_flags(CMAKE_CXX_FLAGS_MINSIZEREL "${CMAKE_CXX_FLAGS_MINSIZEREL}")
strip_unsupported_flags(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}")
strip_unsupported_flags(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS}")
strip_unsupported_flags(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS}")
strip_unsupported_flags(CMAKE_STATIC_LINKER_FLAGS "${CMAKE_STATIC_LINKER_FLAGS}")

option(DISCAMB_BUILD_DOCS "Build the documentation" OFF)
option(DISCAMB_BUILD_EXAMPLES "Build the example programs" OFF)
if((CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang") OR (CMAKE_CXX_COMPILER_ID STREQUAL "Clang"))
    option(USE_STD_SPH_BESSEL "use std::sph_bessel" OFF)
else()
    option(USE_STD_SPH_BESSEL "use std::sph_bessel" ON)
endif()

if(USE_STD_SPH_BESSEL)
set(HAS_SPH_BESSEL true)
endif()


find_package(OpenMP)

# Configure OpenMP GPU offloading flags if OpenMP is found
if(OpenMP_FOUND)
  add_definitions(-DUSE_GPU_OFFLOAD)

  if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    # GCC with nvptx offloading: only add the offload switches (do not forward host-only security/LTO flags)
    # CRITICAL: Add -fno-lto to disable LTO and avoid CF protection issues
    set(_omf "-fopenmp -foffload=-lm -fno-lto")
    # Strip unsupported flags from global flags before combining
    strip_unsupported_flags(_clean_CXX_FLAGS CMAKE_CXX_FLAGS)
    set(CMAKE_CXX_FLAGS "${_clean_CXX_FLAGS} ${_omf}")
    # Also disable LTO in linker flags
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fno-lto")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fno-lto")
    set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -fno-lto")
    set(CMAKE_STATIC_LINKER_FLAGS "${CMAKE_STATIC_LINKER_FLAGS}")
  elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang" OR CMAKE_CXX_COMPILER_ID STREQUAL "AppleClang")
    set(_omf "-fopenmp -fopenmp-targets=nvptx64-nvidia-cuda")
    strip_unsupported_flags(_clean_CXX_FLAGS CMAKE_CXX_FLAGS)
    set(CMAKE_CXX_FLAGS "${_clean_CXX_FLAGS} ${_omf}")
  elseif(CMAKE_CXX_COMPILER_ID STREQUAL "IntelLLVM")
    set(_omf "-fopenmp -fopenmp-targets=spir64")
    strip_unsupported_flags(_clean_CXX_FLAGS CMAKE_CXX_FLAGS)
    set(CMAKE_CXX_FLAGS "${_clean_CXX_FLAGS} ${_omf}")
  endif()

  # Also ensure linker flags do not keep -flto or -fcf-protection
  strip_unsupported_flags(_clean_EXE_LINKER_FLAGS CMAKE_EXE_LINKER_FLAGS)
  set(CMAKE_EXE_LINKER_FLAGS "${_clean_EXE_LINKER_FLAGS}")

  strip_unsupported_flags(_clean_SHARED_LINKER_FLAGS CMAKE_SHARED_LINKER_FLAGS)
  set(CMAKE_SHARED_LINKER_FLAGS "${_clean_SHARED_LINKER_FLAGS}")

  strip_unsupported_flags(_clean_MODULE_LINKER_FLAGS CMAKE_MODULE_LINKER_FLAGS)
  set(CMAKE_MODULE_LINKER_FLAGS "${_clean_MODULE_LINKER_FLAGS}")

  strip_unsupported_flags(_clean_STATIC_LINKER_FLAGS CMAKE_STATIC_LINKER_FLAGS)
  set(CMAKE_STATIC_LINKER_FLAGS "${_clean_STATIC_LINKER_FLAGS}")
endif()


set(CMAKE_DEBUG_POSTFIX "d")
set(CMAKE_MINSIZEREL_POSTFIX "s")
set(CMAKE_RELWITHDEBINFO_POSTFIX "rd")


SET(DISCAMB_INCLUDE_PATH "" CACHE PATH "dir with discamb headers folder 'discamb'")
include_directories(${DISCAMB_INCLUDE_PATH})

if(WIN32)
    SET(CMAKE_DETECTED_WIN32 TRUE)
endif(WIN32)

# add the binary tree to the search path for include files
# so that we will find config.h

include_directories("${PROJECT_BINARY_DIR}/build/include")
INCLUDE_DIRECTORIES(BEFORE include third-party)
ADD_SUBDIRECTORY("${CMAKE_CURRENT_SOURCE_DIR}/src")


IF(DISCAMB_BUILD_EXAMPLES)
    ADD_SUBDIRECTORY("${CMAKE_CURRENT_SOURCE_DIR}/examples")
ENDIF(DISCAMB_BUILD_EXAMPLES)


IF(DISCAMB_BUILD_DOCS)
    ADD_SUBDIRECTORY("${CMAKE_CURRENT_SOURCE_DIR}/doc")
ENDIF(DISCAMB_BUILD_DOCS)

#------ INSTALLATION ------

    IF(INSTALLATION_PREFIX) 
        SET(CMAKE_INSTALL_PREFIX  ${INSTALLATION_PREFIX})
    ELSE(INSTALLATION_PREFIX) 
        IF(CMAKE_SYSTEM_NAME STREQUAL "Windows")
            SET(CMAKE_INSTALL_PREFIX  ${CMAKE_BINARY_DIR}/build/)
        ENDIF(CMAKE_SYSTEM_NAME STREQUAL "Windows")                    
    ENDIF(INSTALLATION_PREFIX) 
    
    install(DIRECTORY include/discamb DESTINATION include)
    install(FILES "${PROJECT_BINARY_DIR}/build/include/discamb/config.h" DESTINATION include/discamb)

#--------

configure_file ("${PROJECT_SOURCE_DIR}/config.h.in" "${PROJECT_BINARY_DIR}/build/include/discamb/config.h")

set (DISCAMB_BUILD_DIR "${PROJECT_BINARY_DIR}/build")