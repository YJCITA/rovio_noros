cmake_minimum_required (VERSION 2.6)
project(rovio)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x -march=native")

##################### Option #####################
OPTION(COMPILE_DEBUG_SWITCH "switch for copmile debug OR release" OFF)
IF (COMPILE_DEBUG_SWITCH)
    MESSAGE(STATUS "copmile debug")
    SET(CMAKE_BUILD_TYPE "Debug")
#     SET(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g -ggdb")
ELSE()
	SET(CMAKE_BUILD_TYPE "Release")
    MESSAGE(STATUS "copmile release")
ENDIF()

OPTION(MAKE_SCENE_SWITCH "switch for MAKE_SCENE" ON)
IF (MAKE_SCENE_SWITCH)
    MESSAGE(STATUS "set MAKE_SCENE true")
    set(MAKE_SCENE TRUE)
ELSE()
	MESSAGE(STATUS "set MAKE_SCENE false")
	set(MAKE_SCENE FALSE)
ENDIF()

set(ROVIO_NMAXFEATURE 25 CACHE STRING "Number of features for ROVIO")
set(ROVIO_NCAM 1 CACHE STRING "Number of enabled cameras")
set(ROVIO_NLEVELS 4 CACHE STRING "Number of image leavels for the features")
set(ROVIO_PATCHSIZE 6 CACHE STRING "Size of patch (edge length in pixel)")
set(ROVIO_NPOSE 0 CACHE STRING "Additional estimated poses for external pose measurements")

add_definitions(-DROVIO_NMAXFEATURE=${ROVIO_NMAXFEATURE})
add_definitions(-DROVIO_NCAM=${ROVIO_NCAM})
add_definitions(-DROVIO_NLEVELS=${ROVIO_NLEVELS})
add_definitions(-DROVIO_PATCHSIZE=${ROVIO_PATCHSIZE})
add_definitions(-DROVIO_NPOSE=${ROVIO_NPOSE})

add_subdirectory(./lightweight_filtering)
#Add custom (non compiling) targets so launch scripts and python files show up in QT Creator's project view.
file(GLOB_RECURSE EXTRA_FILES */*)
add_custom_target(${PROJECT_NAME}_OTHER_FILES ALL WORKING_DIRECTORY ${PROJECT_SOURCE_DIR} SOURCES ${EXTRA_FILES})


##################### Find package #####################
set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
if(0)
	find_package(OpenMP REQUIRED)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

#find_package(Eigen3 REQUIRED)
include(/home/yj/bak/packet/eigen3/script_eigen.cmake REQUIRED)  
include_directories(${EIGEN3_INCLUDE_DIR})

SET(OpenCV_DIR "/home/yj/bak/packet/opencv/opencv3.1/install/share/OpenCV")
MESSAGE(STATUS "set opencv lib: ${OpenCV_DIR}")
FIND_PACKAGE(OpenCV REQUIRED)

find_package(Boost COMPONENTS filesystem system thread REQUIRED)

SET(Pangolin_DIR "/home/yj/bak/packet/Pangolin/install/lib/cmake/Pangolin")
find_package(Pangolin REQUIRED)

find_package(PkgConfig)

find_package(yaml_cpp_catkin QUIET)
if(0 AND ${yaml_cpp_catkin_FOUND})
  message(STATUS "Found yaml_cpp_catkin, using instead of system library.")
  set(YamlCpp_LIBRARIES ${yaml_cpp_catkin_LIBRARIES})
  set(YamlCpp_INCLUDE_DIRS ${yaml_cpp_catkin_INCLUDE_DIRS})
else()
  message(STATUS "No yaml_cpp_catkin, using yaml-cpp system library instead.")
  pkg_check_modules(YamlCpp REQUIRED yaml-cpp>=0.5)
endif()

##################### Include dirs #####################
INCLUDE_DIRECTORIES(
  include
  ./lightweight_filtering/include/
  ${EIGEN3_INCLUDE_DIR}
  ${OpenCV_INCLUDE_DIRS}
  ${Boost_INCLUDE_DIRS}
  ${Pangolin_INCLUDE_DIRS}
  ${YamlCpp_INCLUDE_DIRS}
)

if(MAKE_SCENE)
	message(STATUS "Building ROVIO with openGL Scene Visualization")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DMAKE_SCENE=1")
	find_package(GLEW REQUIRED)
	include_directories(${GLEW_INCLUDE_DIRS})

	find_package(GLUT REQUIRED)
	include_directories(${GLUT_INCLUDE_DIRS})
	link_directories(${GLUT_LIBRARY_DIRS})
	add_definitions(${GLUT_DEFINITIONS})

	find_package(OpenGL REQUIRED)
	include_directories(${OpenGL_INCLUDE_DIRS})
	link_directories(${OpenGL_LIBRARY_DIRS})
	add_definitions(${OpenGL_DEFINITIONS})
	
    add_library(${PROJECT_NAME}  src/Camera.cpp src/Struct_definition.cpp src/FeatureCoordinates.cpp src/FeatureDistance.cpp src/Scene.cpp)
else()
    add_library(${PROJECT_NAME}  src/Camera.cpp src/Struct_definition.cpp src/FeatureCoordinates.cpp src/FeatureDistance.cpp)
endif()

##################### Link #####################
target_link_libraries(${PROJECT_NAME} 
	${YamlCpp_LIBRARIES} 
	${OpenMP_EXE_LINKER_FLAGS} 
	${OPENGL_LIBRARIES} 
	${GLUT_LIBRARY} 
	${GLEW_LIBRARY}
	${OpenCV_LIBS}
	${Boost_LIBRARIES}
	${Pangolin_LIBRARIES}
)

add_executable(rovio_pipeline src/rovio_pipeline.cpp)
target_link_libraries(rovio_pipeline ${PROJECT_NAME})

#add_executable(rovio_node src/rovio_node.cpp)
#target_link_libraries(rovio_node ${PROJECT_NAME})

#add_executable(rovio_rosbag_loader src/rovio_rosbag_loader.cpp)
#target_link_libraries(rovio_rosbag_loader ${PROJECT_NAME})

#add_executable(feature_tracker_node src/feature_tracker_node.cpp)
#target_link_libraries(feature_tracker_node ${PROJECT_NAME})

# add_executable(test_pose_init test/test_pose_init.cpp)
# target_link_libraries(test_pose_init ${PROJECT_NAME})

# -YJ-
# if( EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/gtest/")
# 	message(STATUS "Building GTests!")
# 	option(BUILD_GTEST "build gtest" ON)
# 	#add_subdirectory(gtest gtest)
# 	enable_testing()
# 	include_directories(${gtest_SOURCE_DIR}/include ${gtest_SOURCE_DIR})
# 	add_executable(test_patch src/test_patch.cpp src/Camera.cpp src/FeatureCoordinates.cpp src/FeatureDistance.cpp)
# 	target_link_libraries(test_patch gtest_main gtest pthread 
#         ${YamlCpp_LIBRARIES}
# 	${OpenMP_EXE_LINKER_FLAGS} 
# 	${OPENGL_LIBRARIES} 
# 	${GLUT_LIBRARY} 
# 	${GLEW_LIBRARY}
# 	${OpenCV_LIBS}
# 	${Boost_LIBRARIES}
# 	${Pangolin_LIBRARIES})
# 	add_test(test_patch test_patch)
# 	add_executable(test_mlp src/test_mlp.cpp src/Camera.cpp src/FeatureCoordinates.cpp src/FeatureDistance.cpp)
# 	target_link_libraries(test_mlp gtest_main gtest pthread 
#         ${YamlCpp_LIBRARIES}
# 	${OpenMP_EXE_LINKER_FLAGS} 
# 	${OPENGL_LIBRARIES} 
# 	${GLUT_LIBRARY} 
# 	${GLEW_LIBRARY}
# 	${OpenCV_LIBS}
# 	${Boost_LIBRARIES}
# 	${Pangolin_LIBRARIES})
# 	add_test(test_mlp test_mlp)
# endif()
