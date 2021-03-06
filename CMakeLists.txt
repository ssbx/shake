cmake_minimum_required (VERSION 2.6)
project (SHAKE)


# build PortAudio
set (EXTRA_LIBS "")
set (PA_INCLUDE_DIR "")
set (PA_LIBRARY_DIR "")

if (CMAKE_SYSTEM MATCHES "Windows")

    add_definitions(-DPA_WDMKS_NO_KSGUID_LIB)
    add_subdirectory(deps/portaudio)
    set(PA_INCLUDE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/deps/portaudio/include/)

else()
    # we are on linux/unix/apple use configure script

    if (APPLE)

        find_library(CORE_AUDIO_LIBRARY CoreAudio)
        find_library(AUDIO_TOOLBOX_LIBRARY AudioToolbox)
        find_library(AUDIO_UNIT_LIBRARY AudioUnit)
        find_library(CARBON_LIBRARY Carbon)
        set(EXTRA_LIBS
            ${CORE_AUDIO_LIBRARY}
            ${AUDIO_TOOLBOX_LIBRARY}
            ${AUDIO_UNIT_LIBRARY}
            ${CARBON_LIBRARY})

    else()

        find_library(ALSA REQUIRED)
        set(EXTRA_LIBS -lrt -lm -lasound -ljack -pthread)

    endif()

    include (ExternalProject)
    ExternalProject_Add(ext_portaudio
        SOURCE_DIR          ${CMAKE_CURRENT_SOURCE_DIR}/deps/portaudio
        PREFIX              ${CMAKE_BINARY_DIR}/portaudio
        CONFIGURE_COMMAND   <SOURCE_DIR>/configure
        BUILD_COMMAND       make
        INSTALL_COMMAND     echo Skipping install step for portaudio
        LOG_CONFIGURE       1
        LOG_BUILD           1)
    ExternalProject_Get_Property(ext_portaudio BINARY_DIR)
    ExternalProject_Get_Property(ext_portaudio SOURCE_DIR)
    set(PA_LIBRARY_DIR "${BINARY_DIR}/lib/.libs")
    set(PA_INCLUDE_DIR "${SOURCE_DIR}/include")
    add_library (portaudio STATIC IMPORTED)
    set_property(
        TARGET portaudio
        PROPERTY IMPORTED_LOCATION "${PA_LIBRARY_DIR}/libportaudio.a")

endif(CMAKE_SYSTEM MATCHES "Windows")



# build SHAKE
include_directories (
    . 
    ${PA_INCLUDE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/deps/libwave)
add_library (shake shake.c shake.h)
if (CMAKE_SYSTEM MATCHES "Windows")
    target_link_libraries(shake portaudio_static ${EXTRA_LIBS})
    add_dependencies (shake portaudio_static)
else()
    target_link_libraries(shake portaudio ${EXTRA_LIBS})
    add_dependencies (shake portaudio ext_portaudio)
endif()


# tests
add_executable (shake_example
    shake_example.c
    shake.c
    shake.h)
target_link_libraries(shake_example shake)
add_dependencies(shake_example shake)

# doc
find_package (Doxygen)

if (DOXYGEN_FOUND)

  configure_file (
    ${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in
    ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile @ONLY)

  add_custom_target (shake_doc
    ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
    WORKING DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Generating API documentation with Doxygen" VERBATIM)

endif (DOXYGEN_FOUND)
