cmake_minimum_required(VERSION 2.8)
 
project(Skyscraper)
 
if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
  set (CMAKE_CXX_FLAGS "-fPIC -DBT_USE_DOUBLE_PRECISION ${CMAKE_CXX_FLAGS}")
endif ()

if(APPLE)
	set(CMAKE_MACOSX_RPATH OFF)
endif()

if(UNIX)
	set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/tools/cmake;${CMAKE_MODULE_PATH}")
	if(NOT APPLE)
		set(FRONTENDGLINC "-lGL")
	endif()
endif(UNIX)
 
if (CMAKE_BUILD_TYPE STREQUAL "")
  # CMake defaults to leaving CMAKE_BUILD_TYPE empty. This screws up
  # differentiation between debug and release builds.
  set(CMAKE_BUILD_TYPE "RelWithDebInfo" CACHE STRING "Choose the type of build, options are: None (CMAKE_CXX_FLAGS or CMAKE_C_FLAGS used) Debug Release RelWithDebInfo MinSizeRel." FORCE)
endif ()
 
find_package(OGRE REQUIRED)
 
if(NOT OGRE_FOUND)
  message(SEND_ERROR "Failed to find OGRE.")
endif()
 
set (OGRE_LIBRARIES ${OGRE_LIBRARIES} ${OGRE_Overlay_LIBRARIES})

#find wxWidgets
find_package(wxWidgets COMPONENTS core base gl REQUIRED)

#find FMOD
find_package(FMOD REQUIRED)

#find GTK
if(NOT APPLE)
find_package(GTK2 COMPONENTS gtk REQUIRED)
endif ()

#find BULLET
find_package(Bullet REQUIRED)

#find Caelum
find_package(Caelum REQUIRED)

file(GLOB FRONTEND_FILES
	src/frontend/*.cpp
)
 
file(GLOB SBS_FILES
	src/sbs/*.cpp
)

file(GLOB OGREBULLET_COLL_FILES
	ogrebullet/Collisions/src/*.cpp
	ogrebullet/Collisions/src/Debug/*.cpp
	ogrebullet/Collisions/src/Shapes/*.cpp
	ogrebullet/Collisions/src/Utils/*.cpp
)

file (GLOB OGREBULLET_DYN_FILES
	ogrebullet/Dynamics/src/*.cpp
	ogrebullet/Dynamics/src/Constraints/*.cpp
	ogrebullet/Dynamics/src/Prefab/*.cpp
)

include_directories( ${OIS_INCLUDE_DIRS}
	${OGRE_INCLUDE_DIRS}
	${OGRE_Overlay_INCLUDE_DIRS}
	${FMOD_INCLUDE_DIR}
	${GTK2_INCLUDE_DIRS}
	${BULLET_INCLUDE_DIRS}
	${Caelum_INCLUDE_DIRS}
	src/sbs
	ogrebullet/Collisions/include
	ogrebullet/Collisions/include/Debug
	ogrebullet/Collisions/include/Shapes
	ogrebullet/Collisions/include/Utils
	ogrebullet/Dynamics/include
	ogrebullet/Dynamics/include/Constraints
	ogrebullet/Dynamics/include/Prefab
)
 
include(${wxWidgets_USE_FILE})

#ogrebullet libraries
add_library(OgreBulletCol SHARED ${OGREBULLET_COLL_FILES})
add_library(OgreBulletDyn SHARED ${OGREBULLET_DYN_FILES})

#SBS library
add_library(SBS SHARED ${SBS_FILES})

#frontend executable
add_executable(skyscraper ${FRONTEND_FILES})
 
target_link_libraries(OgreBulletCol ${OGRE_LIBRARIES} ${BULLET_LIBRARIES})
target_link_libraries(OgreBulletDyn OgreBulletCol ${OGRE_LIBRARIES} ${BULLET_LIBRARIES})

target_link_libraries(SBS OgreBulletCol OgreBulletDyn ${OGRE_LIBRARIES} ${OIS_LIBRARIES} ${FMOD_LIBRARY})

target_link_libraries(skyscraper SBS ${OGRE_LIBRARIES} ${OIS_LIBRARIES} ${wxWidgets_LIBRARIES} ${FMOD_LIBRARY} ${GTK2_LIBRARIES} ${Caelum_LIBRARIES} ${FRONTENDGLINC})

if(APPLE)

	set(APP_BUNDLE_CONTENTS_DIR "${CMAKE_BINARY_DIR}/${PROJECT_NAME}.app/Contents")
	set(APP_BUNDLE_LIBRARIES_DIR "${APP_BUNDLE_CONTENTS_DIR}/Frameworks")
	#set(CMAKE_INSTALL_PREFIX "${CMAKE_BINARY_DIR}")
	set(CMAKE_MACOSX_RPATH ON)

	file(REMOVE ${APP_BUNDLE_CONTENTS_DIR}/MacOS/${PROJECT_NAME})
	install(TARGETS skyscraper 
	RUNTIME DESTINATION ${APP_BUNDLE_CONTENTS_DIR}/MacOS)
	#install(TARGETS OgreBulletCol OgreBulletDyn SBS LIBRARY DESTINATION ${APP_BUNDLE_LIBRARIES_DIR} ARCHIVE DESTINATION ${APP_BUNDLE_LIBRARIES_DIR})

	string(REGEX MATCHALL "-l[^;]+" wxWidgets_LIBRARY_FILENAMES "${wxWidgets_LIBRARIES}")
	string(REGEX REPLACE "-l" "${wxWidgets_LIBRARY_DIRS}/lib" wxWidgets_LIBRARY_FILENAMES "${wxWidgets_LIBRARY_FILENAMES}")

	list(TRANSFORM wxWidgets_LIBRARY_FILENAMES APPEND ".dylib")

	set(SKYSCRAPER_OGRE_DEPENDENCIES "${OGRE_LIBRARY_REL};${OGRE_Overlay_LIBRARY_REL};${OGRE_RenderSystem_GL_LIBRARY_REL};${OGRE_Plugin_OctreeSceneManager_LIBRARY_REL};${OGRE_RTShaderSystem_LIBRARY_REL}")
	set(SKYSCRAPER_LIB_DEPENDENCIES "${FMOD_LIBRARY};${Caelum_LIBRARY};${wxWidgets_LIBRARY_FILENAMES}")

	if(OGRE_Plugin_CgProgramManager_FOUND)
		string(APPEND SKYSCRAPER_OGRE_DEPENDENCIES ";${OGRE_Plugin_CgProgramManager_LIBRARY_REL}")
	endif()

	install(CODE "

		file(REMOVE_RECURSE ${APP_BUNDLE_LIBRARIES_DIR})

		execute_process( 
			COMMAND mkdir -p ${APP_BUNDLE_LIBRARIES_DIR}
			COMMAND cp -RHf ${CMAKE_BINARY_DIR}/tools/mac/Skyscraper.app ${CMAKE_BINARY_DIR}
			COMMAND cp -RHf ${SKYSCRAPER_LIB_DEPENDENCIES};${SKYSCRAPER_OGRE_DEPENDENCIES} ${APP_BUNDLE_LIBRARIES_DIR}
		)
		
		include(BundleUtilities)
		fixup_bundle(${APP_BUNDLE_CONTENTS_DIR}/MacOS/${PROJECT_NAME} \"\" \"${APP_BUNDLE_LIBRARIES_DIR}\")
	")

	include(CPack)

	#set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-rpath -Wl,../lib")
	#set(LIBRARY_OUTPUT_PATH ${APP_BUNDLE_LIBRARIES_DIR})
	#
	#if(OGRE_Plugin_CgProgramManager_FOUND)
	#	
	#endif()
	#
	#string(REPLACE ";" " " SKYSCRAPER_DEPENDENCIES "${SKYSCRAPER_DEPENDENCIES}")
	#
	#execute_process( 
	#	COMMAND mkdir -p ${APP_BUNDLE_LIBRARIES_DIR}
	#	COMMAND cp -RH ${SKYSCRAPER_DEPENDENCIES} ${APP_BUNDLE_LIBRARIES_DIR}
	#)
	#execute_process(
	#	COMMAND find ${APP_BUNDLE_LIBRARIES_DIR} -mindepth 1 -maxdepth 1
	#	OUTPUT_VARIABLE SKYSCRAPER_DEPENDENCIES
	#)
	#add_custom_command(TARGET skyscraper
	#	POST_BUILD COMMAND
	#	install_name_tool -add_rpath "@executable_path/../lib/" "${APP_BUNDLE_CONTENTS_DIR}/MacOS/${PROJECT_NAME}"
	#)
	#
	#string(REGEX REPLACE "\n" ";" SKYSCRAPER_DEPENDENCIES "${SKYSCRAPER_DEPENDENCIES}")
	#target_link_libraries(skyscraper SBS ${SKYSCRAPER_DEPENDENCIES})
endif(APPLE)