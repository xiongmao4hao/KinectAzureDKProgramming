cmake_minimum_required(VERSION 2.8.3)
project(K4AProgramming)

# set(OpenCV_DIR C:/Users/zz/Desktop/tryOpencv/OpenCV-MinGW-Build/x64/mingw/lib)
#set(GLUT_INCLUDE_DIR "C:/Program Files (x86)/freeglut/include")
#set(OPENGL_LIBRARY_DIR "C:\\Program Files (x86)\\freeglut\\lib")
#set(Eigen3_DIR "C:/Program Files (x86)/Eigen3/share/eigen3/cmake")
# set(Aruco_DIR "C:\\Program Files (x86)\\aruco\\share\\aruco")

set(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)

set(CMAKE_MODULE_PATH PATHS "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

macro(process_ext_sdk)
  message(STATUS "K4A SDK found in ext/sdk!")
  message(STATUS "!! Not searching for SDK in system path !!")

  # mark that the K4A SDK will need to be copied to the output binary folder
  # this is needed even on Linux if the SDK is not installed to the system path
  set(K4A_INSTALL_NEEDED true)

  get_target_property(K4A_INCLUDE_DIRS k4a::k4a INTERFACE_INCLUDE_DIRECTORIES)
  get_target_property(K4A_CONFIGS k4a::k4a IMPORTED_CONFIGURATIONS)
  message("Configs: ${K4A_CONFIGS}")

  # TODO: if we find more than one configuration, we should fail
  # TODO: potentially clean this logic up
  foreach(imported_config ${K4A_CONFIGS})
    if (${CMAKE_SYSTEM_NAME} STREQUAL "Windows")
      get_target_property(K4A_CONFIG_LIBS k4a::k4a IMPORTED_IMPLIB_${imported_config})
      get_target_property(K4A_CONFIG_DLLS k4a::k4a IMPORTED_LOCATION_${imported_config})
      list(APPEND K4A_LIBS ${K4A_CONFIG_LIBS})
      list(APPEND K4A_DLL_FILES ${K4A_CONFIG_DLLS})
    elseif(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
      get_target_property(K4A_CONFIG_LIBS k4a::k4a IMPORTED_LOCATION_${imported_config})
      get_target_property(K4A_CONFIG_DLLS k4a::k4a IMPORTED_LOCATION_${imported_config})
      list(APPEND K4A_LIBS ${K4A_CONFIG_LIBS})
      list(APPEND K4A_DLL_FILES ${K4A_CONFIG_DLLS})
    endif()
  endforeach()

  if(DEFINED DEPTHENGINE_DLL)
    ## Find the depth engine DLL
    file(GLOB_RECURSE DEPTHENGINE_DLL_FILE
      "${CMAKE_CURRENT_SOURCE_DIR}/ext/sdk/*/${DEPTHENGINE_DLL}" )
    list(APPEND K4A_DLL_FILES ${DEPTHENGINE_DLL_FILE})
  endif()
endmacro()

# Define the names of some files that we are going to try and find
if (${CMAKE_SYSTEM_NAME} STREQUAL "Windows")
  set(DEPTHENGINE_DLL "depthengine_2_0.dll")
elseif(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
  set(DEPTHENGINE_DLL "libdepthengine.so.2.0")
else()
  message(FATAL_ERROR "Platform ${CMAKE_SYSTEM_NAME} is not supported")
endif()

message("Finding K4A SDK binaries")

unset(k4a_DIR CACHE)
unset(azure-kinect-sensor-sdk_DIR CACHE)

if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
  message("Linux mode: Searching for K4A in system path and ./ext/sdk")

  # Test if we can find the k4a library in the system path
  find_package(k4a 1.2.0 QUIET)

  if (${k4a_FOUND})
    message("K4A SDK found in system libraries!")
    # For system libraries on linux, we can just append the target name (k4a::k4a)
    # to what will be passed into target_link_libraries()
    list(APPEND K4A_LIBS k4a::k4a)
  else()
    message("K4A SDK not found in system libraries. Searching ./ext/sdk...")
    find_package(k4a 1.2.0 PATHS "${CMAKE_CURRENT_SOURCE_DIR}/ext/sdk")

    if(${k4a_FOUND})
      set(K4A_INSTALL_NEEDED true)
      message("K4A SDK found in ./ext/sdk!")
      process_ext_sdk()
    else()
      message(FATAL_ERROR "K4A SDK not found in system libraries or ./ext/sdk. Please install the Azure Kinect SDK.")
    endif()

  endif()

elseif(${CMAKE_SYSTEM_NAME} STREQUAL "Windows")
  # Windows will always need K4A install for all targets
  set(K4A_INSTALL_NEEDED true)

  # Try to find K4A installed to Program Files
  find_package(azure-kinect-sensor-sdk 1.2.0 EXACT QUIET PATHS "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

  if (${azure-kinect-sensor-sdk_FOUND})
    message("K4A SDK found in Program Files!")

    list(APPEND K4A_LIBS ${azure-kinect-sensor-sdk_LIBRARIES})
    list(APPEND K4A_BINS ${azure-kinect-sensor-sdk_BINARIES})
    set(K4A_INCLUDE_DIRS ${azure-kinect-sensor-sdk_INCLUDE_DIRS})

    # Create a list of all K4A DLLs
    list(APPEND K4A_DLLS ${azure-kinect-sensor-sdk_LIBRARIES})
    list(APPEND K4A_DLLS ${azure-kinect-sensor-sdk_BINARIES})

    foreach(DLL ${K4A_DLLS})
      set(prop "PROPERTY-NOTFOUND")
      get_property(prop TARGET ${DLL} PROPERTY IMPORTED_LOCATION)
      message(STATUS "IMPORTED_LOCATION: ${prop}")

      if((NOT prop) OR (prop EQUAL "PROPERTY-NOTFOUND"))
        message(FATAL_ERROR "Target '${DLL}' in package azure-kinect-sensor-sdk does not contain an IMPORTED_LOCATION property")
      endif()

      list(APPEND K4A_DLL_FILES ${prop})
    endforeach(DLL)
  else()
    # Try to find K4A installed to ./ext/sdk
    find_package(k4a 1.2.0 QUIET PATHS "${CMAKE_CURRENT_SOURCE_DIR}/ext/sdk")

    if (${k4a_FOUND})
      message("K4A SDK found in ./ext/sdk!")
      process_ext_sdk()
    else()
      message(FATAL_ERROR "K4A SDK not found in Program Files or ./ext/sdk. Please install the Azure Kinect SDK.")
    endif()
  endif()  
endif()

message("K4A Include Dirs: ${K4A_INCLUDE_DIRS}")
message("K4A Libs: ${K4A_LIBS}")
message("K4A DLLs: ${K4A_DLL_FILES}")
message("K4A Install Needed: ${K4A_INSTALL_NEEDED}")

if (${K4A_INSTALL_NEEDED})
  # Tell cmake that we need to reconfigure if any of the DLL files change
  set_property(DIRECTORY APPEND PROPERTY CMAKE_CONFIGURE_DEPENDS ${K4A_DLL_FILES})

  # We need to copy the DLLs into the CATKIN_PACKAGE_LIB_DESTINATION so
  # the node executable can find them on launch, and CATKIN_PACKAGE_BIN_DESTINATION
  # so the nodelet can find them on launch
  set(DLL_COPY_DIRECTORY "${PROJECT_BINARY_DIR}/bin/Release")

  foreach(DIRECTORY ${DLL_COPY_DIRECTORY})
    file(MAKE_DIRECTORY "${DIRECTORY}")
  endforeach(DIRECTORY)

  foreach(DLL ${K4A_DLL_FILES})
    foreach(DIRECTORY ${DLL_COPY_DIRECTORY})
      file(COPY "${DLL}" DESTINATION "${DIRECTORY}")
      get_filename_component(DLL_NAME ${DLL} NAME)
      message(STATUS "Copied dll from ${DLL_NAME} to ${DIRECTORY}")
      # Tell cmake that we need to clean up these DLLs on a "make clean"
      set_property(DIRECTORY APPEND PROPERTY ADDITIONAL_MAKE_CLEAN_FILES "${DIRECTORY}/${DLL_NAME}")
    endforeach(DIRECTORY)
  endforeach(DLL)
endif()

FIND_PACKAGE(OpenCV REQUIRED)
FIND_PACKAGE(GLUT REQUIRED)
# FIND_PACKAGE(PCL 1.9.1 REQUIRED COMPONENTS io visualization)
FIND_PACKAGE(Eigen3 REQUIRED)
set(EIGEN_INCLUDE_DIR ${EIGEN3_INCLUDE_DIR})#为了支持原作者的eigen命名
if (POLICY CMP0072)
  set(OpenGL_GL_PREFERENCE LEGACY)
endif()
find_package(OpenGL REQUIRED)
#以下3个添加均对windows无效
include_directories(${OpenGL_INCLUDE_DIRS})
link_directories(${OpenGL_LIBRARY_DIRS})
add_definitions(${OpenGL_DEFINITIONS})
if(NOT OPENGL_FOUND)
    message(ERROR " OPENGL not found!")
endif(NOT OPENGL_FOUND)


# FIND_PACKAGE(Open3D REQUIRED)

# Find GLFW. You have set the GLFW_ROOT path with your environment.
# set(GLFW_ROOT "H:/c++Enviroment/glfw-3.3.2.bin.WIN64/")
# IF(NOT GLFW_ROOT)
#     MESSAGE("ERROR: GLFW_ROOT must be set!")
# ENDIF(NOT GLFW_ROOT)

# FIND_PATH(GLFW_INCLUDE_DIRS DOC "Path to GLFW include directory."
#     NAMES GLFW/glfw3.h
#     PATHS ${GLFW_ROOT}/include)

# FIND_LIBRARY(GLFW_LIBRARIES DOC "Absolute path to GLFW library."
#     NAMES libglfw3dll.a
#     PATHS ${GLFW_ROOT}/lib-mingw-w64)
find_package(glfw3 REQUIRED)
 
#GLEW_VERBOSE是打开部分报错显示
# set(GLEW_VERBOSE true)
find_package(GLEW REQUIRED)

# Find aruco. You have set the aruco_ROOT path with your environment.
# set(ArUco_ROOT "D:tt/aruco/")
# IF(NOT ArUco_ROOT)
#     MESSAGE("ERROR: ArUco_ROOT must be set!")
# ENDIF(NOT ArUco_ROOT)
# FIND_PATH(ArUco_INCLUDE_DIRS DOC "Path to ArUco include directory." NAMES aruco/aruco.h PATHS ${ArUco_ROOT}/include)

# FIND_LIBRARY(ArUco_LIBRARIES DOC "Absolute path to ArUco library."
#     NAMES aruco312.lib
#     PATHS ${ArUco_ROOT}/lib)
FIND_PACKAGE(aruco REQUIRED)

#set(ArUco_LIBS "C:/Program Files (x86)/aruco/bin/libaruco3112.dll")

# Find GLM. You have to set the glm_room path with your environment
set(GLM_DIR "C:/Program Files (x86)/glm/lib/cmake/glm")
set(GLM_ROOT "C:/Program Files (x86)/glm/include/")
IF(NOT GLM_ROOT)
    MESSAGE("ERROR: GLM_ROOT must be set!")
ENDIF(NOT GLM_ROOT)

FIND_PATH(GLM_INCLUDE_DIRS DOC "Path to GLM include directory." NAMES glm/glm.hpp PATHS ${GLM_ROOT})

# # Find libjpeg-turbo.
# set(libjpeg_turbo_ROOT "D:/libjpeg-turbo-2.0.3")
# IF(NOT libjpeg_turbo_ROOT)
#     MESSAGE("ERROR: libjpeg_turbo_ROOT must be set!")
# ENDIF(NOT libjpeg_turbo_ROOT)

# set(libjpeg_turbo_INCLUDE_DIRS ${libjpeg_turbo_ROOT}/include)

# FIND_LIBRARY(libjpeg_turbo_LIBRARIES_jpeg DOC "Absolute path to libjpeg-turbo jpeg library."
#     NAMES jpeg-static.lib 
#     PATHS ${libjpeg_turbo_ROOT}/lib)

# FIND_LIBRARY(libjpeg_turbo_LIBRARIES_turbojpeg DOC "Absolute path to libjpeg-turbo turbo library."
#     NAMES turbojpeg-static.lib
#     PATHS ${libjpeg_turbo_ROOT}/lib)

# # Find libyuv.
# set(libyuv_ROOT "D:/libyuv")
# IF(NOT libyuv_ROOT)
#     MESSAGE("ERROR: libyuv_ROOT must be set!")
# ENDIF(NOT libyuv_ROOT)

# set(libyuv_INCLUDE_DIRS ${libyuv_ROOT}/include)

# FIND_LIBRARY(libyuv_LIBRARIES DOC "Absolute path to libyuv library."
#     NAMES yuv.lib
#     PATHS ${libyuv_ROOT}/lib)

add_subdirectory(OpenCV_OneKinect)
add_subdirectory(OpenCV_TwoKinects)
# add_subdirectory(OpenGL_GLUT_ShowImage)
# add_subdirectory(OpenGL_GLUT_ArUco_AR)
# add_subdirectory(OpenGL_GLFW_GLEW_ArUco_AR)
# add_subdirectory(OpenGL_GLFW_GLEW_PointCloudRenderer)
add_subdirectory(Aruco_TwoKinects_Calibration_Extrinsics)
# add_subdirectory(OpenCV_TwoKinects_GreenScreen)
# add_subdirectory(OneKinect_Recording_RGB_DEPTH_IR)
# add_subdirectory(OneKinect_Playback_RGB_DEPTH_IR)
# add_subdirectory(Open3D_OneKinect)