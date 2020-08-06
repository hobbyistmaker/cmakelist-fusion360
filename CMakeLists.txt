cmake_policy(SET CMP0048 NEW)
cmake_minimum_required(VERSION 3.17)
project (MyProject VERSION 0.1.0.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED True)
set(CMAKE_OSX_DEPLOYMENT_TARGET "10.13" CACHE STRING "Minimum Fusion 360 supported version of MacOS/OSX.")
set(CMAKE_OSX_SYSROOT "/Library/Developer/CommandLineTools/SDKs/MacOSX10.13.sdk")
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

configure_file(src/MyProject.h.in src/MyProject.h)

IF(APPLE)
    set(AUTODESK_FOLDER "$ENV{HOME}/Library/Application\ Support/Autodesk/Autodesk\ Fusion\ 360/API/CPP")
    set(F360_INCLUDES "${AUTODESK_FOLDER}/include")
    set(F360_LIBS "${AUTODESK_FOLDER}/lib")
    set(LOCAL_INCLUDES "/usr/local/include")
ELSEIF(WIN32)
    set(AUTODESK_HASH d2d901e2ef5ae5c384b4ac001a48c4e37589dc43) # Pulled from the Autodesk folder name under your APPDATA folder
    set(AUTODESK_FOLDER "$ENV{LOCALAPPDATA}/Autodesk/webdeploy/production/${AUTODESK_HASH}/CPP")
    set(F360_INCLUDES "${AUTODESK_FOLDER}/include")
    set(F360_LIBS "${AUTODESK_FOLDER}/lib")
ENDIF(APPLE)

if (MSVC_VERSION GREATER_EQUAL "1900") # Make sure that the Visual Studio compiler is using C++17
    include(CheckCXXCompilerFlag)
    CHECK_CXX_COMPILER_FLAG("/std:c++17" _cpp_latest_flag_supported)
    if (_cpp_latest_flag_supported)
        add_compile_options("/std:c++17")
    endif()
endif()

file(GLOB_RECURSE SOURCE_FILES
        LIST_DIRECTORIES true
        src/lib/*.cpp
        src/lib/*.hpp
        src/lib/*.h
        )

message( STATUS "Users home folder: $ENV{HOME}" )
message( STATUS "Autodesk Fusion 360 Includes folder: ${F360_INCLUDES}" )
message( STATUS "Target file destination: ${CMAKE_CURRENT_SOURCE_DIR}" )
message( STATUS "Binary dir: ${CMAKE_BINARY_DIR}" )
message( STATUS "Source dir: ${CMAKE_SOURCE_DIR}" )
message( STATUS "Found sources: ${SOURCE_FILES}" )
IF(WIN32)
    message( STATUS "App data: $ENV{LOCALAPPDATA}")
ENDIF()

link_directories("${F360_LIBS}")

add_library(MyProject SHARED ${SOURCE_FILES})
set_target_properties(MyProject PROPERTIES PREFIX "")
set_target_properties(
        MyProject PROPERTIES
        RUNTIME_OUTPUT_DIRECTORY_DEBUG "${CMAKE_CURRENT_SOURCE_DIR}"
        RUNTIME_OUTPUT_DIRECTORY_RELEASE "${CMAKE_CURRENT_SOURCE_DIR}"
)
target_sources(MyProject
        PRIVATE ${SOURCE_FILES}
        PUBLIC src/MyProject.cpp
        )

include_directories(${CMAKE_SOURCE_DIR}/src)
include_directories(${F360_INCLUDES})

target_include_directories(MyProject PRIVATE "${F360_INCLUDES}")
target_include_directories(MyProject PRIVATE "${F360_INCLUDES}/Core")
target_include_directories(MyProject PRIVATE "${F360_INCLUDES}/Core/UserInterface")

IF(UNIX)
    find_library(ADSK_CORE core.dylib PATHS "${F360_LIBS}/")
    find_library(ADSK_FUSION fusion.dylib PATHS "${F360_LIBS}/")
    find_library(ADSK_CAM cam.dylib PATHS "${F360_LIBS}/")
ENDIF(UNIX)

IF(WIN32)
    find_library(ADSK_CORE core.lib PATHS "${F360_LIBS}/")
    find_library(ADSK_FUSION fusion.lib PATHS "${F360_LIBS}/")
    find_library(ADSK_CAM cam.lib PATHS "${F360_LIBS}/")
ENDIF()

target_link_libraries(MyProject PUBLIC ${ADSK_CORE} ${ADSK_FUSION} ${ADSK_CAM})

if(CMAKE_GENERATOR STREQUAL Xcode)
    configure_file(
            WorkspaceSettings.xcsettings
            ${CMAKE_BINARY_DIR}/${PROJECT_NAME}.xcodeproj/project.xcworkspace/xcuserdata/$ENV{USER}.xcuserdatad/WorkspaceSettings.xcsettings
            COPYONLY
    )
endif()

#file(COPY ${CMAKE_BINARY_DIR}/MyProject.dylib DESTINATION ${CMAKE_SOURCE_DIR})
