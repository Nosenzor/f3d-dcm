cmake_minimum_required(VERSION 3.21)

project(f3d-plugin-open3sdcm LANGUAGES CXX)

include(GNUInstallDirs)

# Only internal build is supported for static plugins
if(PROJECT_IS_TOP_LEVEL)
  message(FATAL_ERROR "open3sdcm plugin cannot be built externally")
endif()

include(f3dPlugin)

# Open3SDCM library
# For development, use local path. For production, consider using FetchContent or git submodule
option(OPEN3SDCM_USE_LOCAL "Use local Open3SDCM path for development" OFF)
set(OPEN3SDCM_LOCAL_SOURCE_DIR "" CACHE PATH "Path to a local Open3SDCM checkout")
if(OPEN3SDCM_USE_LOCAL)
  if(OPEN3SDCM_LOCAL_SOURCE_DIR STREQUAL "")
    message(FATAL_ERROR "OPEN3SDCM_USE_LOCAL=ON requires OPEN3SDCM_LOCAL_SOURCE_DIR to be set")
  endif()

  set(OPEN3SDCM_SOURCE_DIR "${OPEN3SDCM_LOCAL_SOURCE_DIR}")
  set(OPEN3SDCM_LIB_SOURCE_DIR "${OPEN3SDCM_SOURCE_DIR}/Lib")
  if(NOT EXISTS "${OPEN3SDCM_LIB_SOURCE_DIR}/CMakeLists.txt")
    message(FATAL_ERROR "Open3SDCM not found at ${OPEN3SDCM_SOURCE_DIR}")
  endif()

  # Add Open3SDCM as subdirectory
  add_subdirectory(${OPEN3SDCM_LIB_SOURCE_DIR} external/Open3SDCM)

  # Remove unused Boost dependency from Open3SDCM (not actually used in ParseDcm)
  # We'll patch it by removing the boost_dynamic_bitset requirement
  set(Boost_USE_STATIC_LIBS OFF)
  set(Boost_USE_MULTITHREADED ON)
else()
  # For CI/production, use FetchContent
  include(FetchContent)
  FetchContent_Declare(
    Open3SDCM
    URL https://github.com/Nosenzor/Open3SDCM/archive/3f082e64f08fff4833e8fc857185c2cfee6cd302.tar.gz
    URL_HASH SHA256=bb6b7f28b888b39569f8df5cb3477286264fc951a2a123ebe538e65f35e72955
    SOURCE_SUBDIR Lib
  )
  FetchContent_MakeAvailable(Open3SDCM)
  set(OPEN3SDCM_LIB_SOURCE_DIR "${Open3SDCM_SOURCE_DIR}/Lib")
endif()

# Required dependencies for Open3SDCM
# Poco is required for XML parsing and ZIP extraction
find_package(Poco CONFIG REQUIRED COMPONENTS XML JSON Zip)

# OpenSSL is required for CE schema (encrypted files)
find_package(OpenSSL REQUIRED)

# Boost is required by Open3SDCM
find_package(Boost REQUIRED)

# Initialize plugin
f3d_plugin_init()

# Declare the reader/importer for 3Shape DCM files
# Note: .dcm extension is also used by DICOM reader
# We use SCORE 70 (> DICOM's 50) + content-based canRead() to resolve conflict
f3d_plugin_declare_reader(
  NAME Open3SDCM
  EXTENSIONS dcm
  MIMETYPES application/vnd.3shape-dcm
  VTK_IMPORTER vtkF3DOpen3SDCMImporter
  FORMAT_DESCRIPTION "3Shape DCM (Dental CAD Model)"
  SCORE 70
  CAN_READ STATIC
)

# Add Open3SDCM include directory for the generated reader code
list(APPEND F3D_PLUGIN_INCLUDES ${OPEN3SDCM_LIB_SOURCE_DIR}/src)

# Build the plugin
f3d_plugin_build(
  NAME open3sdcm
  VERSION 1.0
  DESCRIPTION "3Shape DCM file format support via Open3SDCM library"
  VTK_MODULES IOGeometry IOImage RenderingCore
  FREEDESKTOP
)

# Link Open3SDCM dependencies to our VTK reader
# This is done automatically by f3d_plugin_build, but we need to ensure
# the Open3SDCM library is linked properly

# Link the plugin to the VTK reader library and Open3SDCM
# This needs to be done after f3d_plugin_build creates the target
target_link_libraries(f3d-plugin-open3sdcm PRIVATE
  f3d::vtkextOpen3SDCM
  Open3SDCMLib
  Poco::XML
  Poco::Zip
  OpenSSL::Crypto
  ${Boost_LIBRARIES}
)

# Add Open3SDCM include directory to the plugin target
# This is needed for the custom code (.inl file) which includes ParseDcm.h
target_include_directories(f3d-plugin-open3sdcm
  PRIVATE
    ${OPEN3SDCM_LIB_SOURCE_DIR}/src
)

# Print configuration summary
message(STATUS "Plugin: open3sdcm configured")
message(STATUS "  Open3SDCM source: ${OPEN3SDCM_LIB_SOURCE_DIR}")
message(STATUS "  Poco found: ${Poco_DIR}")
message(STATUS "  OpenSSL found: ${OPENSSL_DIR}")

message(STATUS "  Boost found: ${Boost_DIR}")
