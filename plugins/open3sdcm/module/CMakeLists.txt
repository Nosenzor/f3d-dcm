set(classes
  vtkF3DOpen3SDCMImporter
  )

# Add Open3SDCM include directory to the VTK module
if(DEFINED OPEN3SDCM_LIB_SOURCE_DIR)
  include_directories(${OPEN3SDCM_LIB_SOURCE_DIR}/src)
endif()

vtk_module_add_module(f3d::vtkextOpen3SDCM
  NO_INSTALL
  FORCE_STATIC
  CLASSES ${classes})

vtk_module_set_properties(f3d::vtkextOpen3SDCM CXX_STANDARD 20)
