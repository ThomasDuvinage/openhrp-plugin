# @author Shin'ichiro Nakaoka

option(BUILD_OPENHRP_PLUGIN "Building OpenHRPPlugin" OFF)
option(BUILD_OPENHRP_PLUGIN_FOR_3_0 "Building OpenHRPPlugin for OpenHRP 3.0.x" OFF)
option(BUILD_OPENHRP_MODEL_LOADER "Building OpenHRPModelLoader" OFF)

option(CHOREONOID_PLUGIN_LINK_USE_KEYWORD "Choreonoid plugin should use the PUBLIC link keyword" ON)
if(CHOREONOID_PLUGIN_LINK_USE_KEYWORD)
  set(LINK_KEYWORD PUBLIC)
else()
  set(LINK_KEYWORD)
endif()

# set(CMAKE_BUILD_TYPE Debug)

if(BUILD_OPENHRP_PLUGIN)
  if(NOT BUILD_OPENRTM_PLUGIN)
    message(FATAL_ERROR "OpenHRP Plugin requires OpenRTMPlugin.")
  endif()

  if(BUILD_OPENHRP_PLUGIN_FOR_3_0)
    set(versions 3.1 3.0)
  else()
    set(versions 3.1)
  endif()

  if(UNIX AND NOT APPLE)
    # Is this necessary to use both plugins for version 3.0 and 3.1 at the same time ?
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,-Bsymbolic")
  endif()

  try_compile(HAS_OLD_COLLISION_LINK_PAIR ${CMAKE_CURRENT_BINARY_DIR}/cmake/try-compile
    SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/cmake/detect-old-collision-link-pair.cpp
    COMPILE_DEFINITIONS -I${PROJECT_SOURCE_DIR}/include -I${PROJECT_SOURCE_DIR} -I${EIGEN3_INCLUDE_DIRS}
  )

  add_definitions(${OPENRTM_DEFINITIONS})
  include_directories(${CMAKE_CURRENT_BINARY_DIR} ${OPENRTM_INCLUDE_DIRS})
  link_directories(${OPENRTM_LIBRARY_DIRS})

  foreach(version ${versions})

    #-------------------- Sutbs & Skeletons -------------------------
    set(idls
      OpenHRPCommon
      ModelLoader
      DynamicsSimulator
      CollisionDetector
      ViewSimulator
      Controller
      OnlineViewer
      )

    if(version EQUAL 3.0)
      set(defver "OPENHRP_3_0")
    else()
      set(defver "OPENHRP_3_1")
      set(idls ${idls}
	World
	InterpreterService
	ClockGenerator)
    endif()
    
    set(sources
      OpenHRPControllerBase.cpp
      )
    set(headers
      OpenHRPControllerBase.h
      )

    set(target CnoidOpenHRP${version})

    set(idl_cpp_files "")
    set(idl_h_files "")
    idl_compile_cpp(idl_cpp_files idl_h_files corba/OpenHRP/${version} ${idls})

    add_cnoid_library(${target} STATIC ${sources} ${idl_cpp_files} ${idl_h_files})
    target_compile_definitions(${target} PUBLIC ${defver})
    if(HAS_OLD_COLLISION_LINK_PAIR)
      target_compile_definitions(${target} PUBLIC HAS_OLD_COLLISION_LINK_PAIR)
    endif()
    target_link_libraries(${target} ${LINK_KEYWORD} CnoidCorba)
    apply_common_setting_for_library(${target} "${headers}")

    # Plugin
    set(target CnoidOpenHRP${version}Plugin)

    set(sources
      OpenHRPPlugin.cpp
      DynamicsSimulator_impl.cpp
      OpenHRPControllerItem.cpp
      OnlineViewerServer.cpp
      OpenHRPOnlineViewerItem.cpp
      )

    if(version EQUAL 3.1)
      set(sources ${sources}
	OpenHRPInterpreterServiceItem.cpp
	OpenHRPClockGeneratorItem.cpp
	)
    endif()
  
    set(headers
      exportdecl.h
      OpenHRPInterpreterServiceItem.h
      )
  
    if(MSVC)
      add_compile_options(${OPENRTM_CFLAGS})
    endif()

    make_gettext_mofiles(${target} mofiles)
    add_cnoid_plugin(${target} SHARED ${sources} ${headers} ${mofiles})
    target_compile_definitions(${target} PUBLIC ${defver})

    if(version EQUAL 3.0)
      target_link_libraries(${target} ${LINK_KEYWORD} CnoidBodyPlugin CnoidOpenHRP${version})
    else()
      target_link_libraries(${target} ${LINK_KEYWORD} CnoidOpenRTMPlugin CnoidBodyPlugin CnoidOpenHRP${version})
    endif()

    apply_common_setting_for_plugin(${target} "${headers}")

  endforeach()

  if(ENABLE_PYTHON AND USE_PYBIND11)
    add_subdirectory(pybind11)
  elseif(BUILD_BOOST_PYTHON_MODULES)
    add_subdirectory(boostpython)
  endif()

  add_subdirectory(sample)

endif()

if(BUILD_OPENHRP_MODEL_LOADER)
  
  set(target cnoid-openhrp-model-loader)

  set(sources
    ShapeSetInfo_impl.cpp
    SceneInfo_impl.cpp
    BodyInfo_impl.cpp
    ModelLoader_impl.cpp
    ModelLoaderUtil.cpp
    ModelLoaderServer.cpp
    )

  set(idls
    OpenHRPCommon
    ModelLoader
    ViewSimulator
    World
    )

  set(idl_cpp_files "")
  set(idl_h_files "")
  idl_compile_cpp(idl_cpp_files idl_h_files corba/OpenHRPModelLoader ${idls})
      
  add_cnoid_executable(${target} ${sources} ${idl_cpp_files})
  target_link_libraries(${target} CnoidCorba CnoidBody ${OMNIDYNAMIC_LIBRARIES})
endif()
