cmake_minimum_required(VERSION 2.8.3)

if(MSVC)
    set(CMAKE_CXX_FLAGS_RELEASE "/Z7")
    set(CMAKE_EXE_LINKER_FLAGS_RELEASE "/INCREMENTAL:NO /DEBUG /OPT:REF /OPT:ICF")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /wd4100 /wd4127 /wd4150 /wd4191 /wd4242 /wd4244 /wd4251 /wd4305 /wd4365 /wd4512 /wd4514 /wd4548 /wd4571 /wd4619 /wd4625 /wd4626 /wd4628 /wd4668 /wd4710 /wd4711 /wd4738 /wd4820 /wd4946 /wd4986 /wd4819 /wd4996 /wd4267")
    add_definitions(-DNDEBUG)
	add_definitions(-D_USE_MATH_DEFINES)
    add_definitions(-DNOMINMAX)
endif()
if( MINGW )
    if( CMAKE_BUILD_TYPE STREQUAL "Debug" )
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++2a -O0 -g -Wall -Wunused-function -pthread")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -g -pthread")
        message("MINGW - Debug mode")
    elseif( CMAKE_BUILD_TYPE STREQUAL "Release" ) # debug
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -O2 -pthread -Wunused-function")
        set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -O2 -pthread")
        message("MINGW - Release mode")
    endif() # Debug
    #
    if( PROJECT_ARCHITECTURE STREQUAL "x64" ) # x64
    set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -m64")
    elseif(PROJECT_ARCHITECTURE STREQUAL "x86") # x86
    set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -m32")
    endif() # x64
    #set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} " )
    set( CMAKE_EXECUTABLE_SUFFIX ".exe" )
endif() # MINGW

INCLUDE_DIRECTORIES(${K4A_INCLUDE_DIRS})
INCLUDE_DIRECTORIES(${OpenCV_INCLUDE_DIR})
INCLUDE_DIRECTORIES(${PROJECT_SOURCE_DIR})
add_executable(OpenOneKinect "Source_1.cpp" "../Util.cpp" "../StaticImageProperties.cpp")
target_link_libraries(OpenOneKinect ${OpenCV_LIBS} ${K4A_LIBS})