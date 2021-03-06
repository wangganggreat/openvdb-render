set(PLUGIN_NAME openvdb_render)

add_definitions(-D_BOOL -DREQUIRE_IOSTREAM)

if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
    add_definitions(-DLINUX -pthread)
endif ()

set(Boost_USE_STATIC_LIBS ON)

if (USE_CUDA)
    find_package(CUDA)
endif ()
find_package(Maya)
find_package(Boost COMPONENTS regex)

if (NOT ILMBASE_ROOT)
    set(ILMBASE_ROOT $ENV{ILMBASE_ROOT})
endif ()

find_path(OPENVDB_INCLUDE
    NAMES openvdb/openvdb.h
    HINTS $ENV{OPENVDB_ROOT}/include)
include_directories(${LUMA_NODE_ID_INCLUDE})
include_directories(${OPENVDB_INCLUDE})
include_directories(${Boost_INCLUDE_DIRS})
include_directories(SYSTEM ${ILMBASE_ROOT}/include) #register is deprecated with c++11

include_directories(${MAYA_INCLUDE_DIRS})
link_directories(${MAYA_LIBRARY_DIR})

set(MAYA_LIBS OpenMaya Foundation OpenMayaAnim OpenMayaFX OpenMayaUI OpenMayaRender)

if (WIN32)
    find_library(OPENVDB_STATIC_LIBRARY
        NAMES libopenvdb
        HINTS $ENV{OPENVDB_ROOT}/lib)
else ()
    find_library(OPENVDB_STATIC_LIBRARY NAMES libopenvdb.a)
endif ()
find_package(OpenGL REQUIRED)

file(GLOB SRC *.cpp)

if (USE_CUDA)
    add_definitions(-DUSE_CUDA)
    find_library(CUDA_STATIC_LIBRARIES libcudart_static.a
            HINT $ENV{CUDA_HOME}/lib64)
    set(CUDA_NVCC_FLAGS ${CUDA_NVCC_FLAGS} --compiler-options -fPIC -O3 -std=c++11 -Wno-deprecated-gpu-targets) # pass fpic to the host compiler
    cuda_compile(point_sorter point_sorter.cu)
    cuda_include_directories($ENV{THRUST_ROOT})
    include_directories($ENV{THRUST_ROOT}) # this probably helps clion intellisense
    set(SRC ${SRC} ${point_sorter})
endif ()

if (INCLUDE_HEADERS_IN_BUILD)
    file(GLOB HDR *.hpp *.h ../util/*.hpp ../util/*.h)
    set(SRC ${SRC} ${HDR})
endif ()

add_library(${PLUGIN_NAME} SHARED ${SRC})
set_target_properties(${PLUGIN_NAME} PROPERTIES PREFIX "")
if (WIN32)
    set_target_properties(${PLUGIN_NAME} PROPERTIES SUFFIX ".mll")
endif ()

target_link_libraries(${PLUGIN_NAME} ${OPENGL_gl_LIBRARY} ${MAYA_LIBS} ${OPENVDB_STATIC_LIBRARY} ${Boost_LIBRARIES})
if (USE_CUDA)
    target_link_libraries(${PLUGIN_NAME} ${CUDA_STATIC_LIBRARIES})
endif ()

install(TARGETS ${PLUGIN_NAME}
        DESTINATION plug-ins)

file(GLOB TEMPLATES template/*.py)

install(FILES ${TEMPLATES}
        DESTINATION scripts)
