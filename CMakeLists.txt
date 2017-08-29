#
# CMake listfile to specify the build process, see:
# http://www.cmake.org/cmake/help/documentation.html
#
# Require CMake 3.1+ for OGDF support.
cmake_minimum_required(VERSION 3.1.0 FATAL_ERROR)

project(OGDFPLUGIN)



# add an uninstall target
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in" "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake" IMMEDIATE @ONLY)
add_custom_target(uninstall COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)


# optional features:
option(ENABLE_DEMO "Enable Demo" ON)


# add extra modules:
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules/")

# show the build messages:
set(CMAKE_VERBOSE_MAKEFILE ON)



find_package(Qt5Core 5.1.0 REQUIRED)
find_package(Qt5Gui 5.1.0 REQUIRED)
find_package(Qt5Widgets 5.1.0 REQUIRED)
find_package(Qt5Qml 5.1.0 REQUIRED)


# Find the OGDF library:
find_package(OGDF REQUIRED)
include_directories(${OGDF_INCLUDE_DIRS})

# add the binary directory to include path (for auto-generated headers):
include_directories(${CMAKE_CURRENT_BINARY_DIR})



# set default CXX flags:
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -pedantic -Wextra")

STRING(TOLOWER "${CMAKE_BUILD_TYPE}" CMAKE_BUILD_TYPE_LOWER)
if(CMAKE_BUILD_TYPE_LOWER STREQUAL "debug")
    message("Building in debug mode, enabling debug flags")
    add_definitions(-DDEBUG -D_DEBUG -D_DEBUG_)
endif()

if(APPLE OR UNIX OR MINGW)
    set(RELEASE_FLAGS ${RELEASE_FLAGS} -O3 -march=corei7-avx)
    set(DEBUG_FLAGS ${DEBUG_FLAGS} -O1 -march=corei7-avx -ggdb)
endif()


# Add QML OGDF plug-in target.
set(OGDFPLUGIN_SOURCES ${OGDFPLUGIN_SOURCES}
    ogdfplugin/edgemodel.cpp
    ogdfplugin/graph.cpp
    ogdfplugin/graphlayout.cpp
    ogdfplugin/nodemodel.cpp
    ogdfplugin/ogdfplugin.cpp
    ogdfplugin/GraphView.qml
    ogdfplugin/EdgeSpline.qml)

set(OGDFPLUGIN_HEADERS ${OGDFPLUGIN_HEADERS}
    ogdfplugin/edgemodel.h
    ogdfplugin/graph.h
    ogdfplugin/graphlayout.h
    ogdfplugin/nodemodel.h
    ogdfplugin/ogdfplugin.h)


# Instruct CMake to run moc automatically when needed.
set(CMAKE_AUTOMOC ON)
#qt5_wrap_cpp(MOC_SOURCES ${OGDFPLUGIN_HEADERS})


# create the main target:
add_library(ogdfplugin SHARED ${OGDFPLUGIN_SOURCES} ${OGDFPLUGIN_HEADERS} ${MOC_SOURCES})

target_compile_options(ogdfplugin PRIVATE "$<$<CONFIG:RELEASE>:${RELEASE_FLAGS}>")
target_compile_options(ogdfplugin PRIVATE "$<$<CONFIG:DEBUG>:${DEBUG_FLAGS}>")

# Set required compiler parameters:
set_property(TARGET ogdfplugin PROPERTY CXX_STANDARD 11)
set_property(TARGET ogdfplugin PROPERTY CXX_STANDARD_REQUIRED ON)

# shared libraries need PIC
set_property(TARGET ogdfplugin PROPERTY POSITION_INDEPENDENT_CODE ON)


target_link_libraries(ogdfplugin ${OGDF_LIBRARIES})
qt5_use_modules(ogdfplugin Qml Core)
if(WIN32)
    target_link_libraries(ogdfplugin "Psapi.lib")
endif()
target_link_libraries(ogdfplugin z m stdc++)


# Install target
install(TARGETS ogdfplugin DESTINATION bin/OGDF)
install(FILES ogdfplugin/EdgeSpline.qml DESTINATION bin/OGDF)
install(FILES ogdfplugin/GraphView.qml DESTINATION bin/OGDF)
install(FILES ogdfplugin/qmldir DESTINATION bin/OGDF)



if(ENABLE_DEMO)
    # Add QML OGDF demo target.
    set(DEMO_SOURCES ${DEMO_SOURCES}
        demo/main.cpp)
    set(DEMO_QRC_SOURCES ${DEMO_QRC_SOURCES}
        demo/main.qrc)
    set(DEMO_QML ${DEMO_QML}
        demo/qml/CanvasView.qml
        demo/qml/main.js
        demo/qml/main.qml
        demo/qml/Sidebar.qml)


    qt5_add_resources(DEMO_QRC_SOURCES ${DEMO_QRC_SOURCES})
    add_executable(ogdf-demo ${DEMO_SOURCES} ${DEMO_QRC_SOURCES} ${DEMO_QML})

    set_target_properties(ogdf-demo PROPERTIES PROJECT_LABEL "demo")

    target_compile_options(ogdfplugin PRIVATE "$<$<CONFIG:RELEASE>:${RELEASE_FLAGS}>")
    target_compile_options(ogdfplugin PRIVATE "$<$<CONFIG:DEBUG>:${DEBUG_FLAGS}>")

    # Set required compiler parameters:
    set_property(TARGET ogdfplugin PROPERTY CXX_STANDARD 11)
    set_property(TARGET ogdfplugin PROPERTY CXX_STANDARD_REQUIRED ON)

    add_dependencies(ogdf-demo ogdfplugin)
    qt5_use_modules(ogdf-demo Quick Qml Core Widgets)

    # Install target:
    install(TARGETS ogdf-demo DESTINATION bin)
endif()
