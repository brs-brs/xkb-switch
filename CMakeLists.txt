cmake_minimum_required(VERSION 3.10)

project(XKB-SWITCH)
set(MAJOR_VERSION 2)
set(MINOR_VERSION 0)
set(RELEASE_VERSION 1)
set(XKBSWITCH_VERSION ${MAJOR_VERSION}.${MINOR_VERSION}.${RELEASE_VERSION})
add_definitions(-DXKBSWITCH_VERSION="${XKBSWITCH_VERSION}")

# Check presence of development libraries required for build
find_package(X11 REQUIRED)
if(NOT X11_FOUND)
    message(FATAL_ERROR "Not found development files of 'libx11' required for build. (Install libx11-dev or libx11-devel package.) CMake will exit.")
elseif(NOT X11_Xkbfile_FOUND)
    message(FATAL_ERROR "Not found development files of 'libxkbfile' required for build. (Install libxkbfile-dev or libxkbfile-devel package.) CMake will exit.")
endif()
include_directories(${X11_INCLUDE_DIR})
link_directories(${X11_LIBRARY_DIR})

# Compile and link program
option(BUILD_XKBSWITCH_LIB
    "Build a library compatible with vim's libcall interface" ON)
if(BUILD_XKBSWITCH_LIB)
    set(xkblib xkbswitch)
    add_library(${xkblib} SHARED src/XKbSwitchApi.cpp src/XKeyboard.cpp)
    set_target_properties(${xkblib} PROPERTIES VERSION ${XKBSWITCH_VERSION} SOVERSION ${MAJOR_VERSION})
    target_link_libraries(${xkblib} X11 xkbfile)
    add_executable(xkb-switch src/XKbSwitch.cpp)
    target_link_libraries(xkb-switch ${xkblib})
else()
    add_executable(xkb-switch src/XKbSwitch.cpp src/XKeyboard.cpp)
    target_link_libraries(xkb-switch X11 xkbfile)
endif()

# Install program
install(TARGETS xkb-switch ${xkblib}
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib OPTIONAL
)

set(MAN_COMPRESSION "gzip" CACHE STRING "Manpages compression tool")
set(MANDIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE STRING "Manpages installation path")

# Function to compress and install man page
function(install_man man_filename man_type)
    # check what compression tool is available
    find_program(COMPRESS_EXECUTABLE NAMES ${MAN_COMPRESSION})
    if(NOT COMPRESS_EXECUTABLE)
      set(MAN_COMPRESSION NO)
    endif()
    # set input an output file names
    set(raw_man man/${man_filename}.${man_type})
    # compress if there is the compression tool
    if(MAN_COMPRESSION)
      if(COMPRESS_EXECUTABLE MATCHES "gzip")
          set(installed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type}.gz)
        else()
          set(installed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type}.${MAN_COMPRESSION})
        endif()
        add_custom_command(OUTPUT ${installed_man}
            COMMAND cat ${raw_man} | ${COMPRESS_EXECUTABLE} > ${installed_man}
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            DEPENDS ${raw_man}
            COMMENT "Compressing man file ${raw_man} to ${installed_man}"
        )
    # elsewise just copy
    else()
        set(installed_man ${CMAKE_BINARY_DIR}/${man_filename}.${man_type})
        message(WARNING "There is no compression tool for man pages. Not compressed copy of man file will be used.")
        add_custom_command(OUTPUT ${installed_man}
            COMMAND cp ${raw_man} ${CMAKE_BINARY_DIR}
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            DEPENDS ${raw_man}
            COMMENT "Copying man file from ${raw_man}."
        )
    endif()
    # add actions
    add_custom_target(man_${man_filename}_${man_type} ALL DEPENDS ${installed_man})
    install(FILES ${installed_man}
        DESTINATION ${MANDIR}/man${man_type}
    )
endfunction()

# Compress and install man page
install_man(xkb-switch 1)

# Include CPack
include(CPack)

# Set CPack variables
set(CPACK_PACKAGE_NAME "xkb-switch")
set(CPACK_PACKAGE_VERSION ${XKBSWITCH_VERSION})
set(CPACK_PACKAGE_CONTACT "brs@brs.im")

# Set Debian-specific variables
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Sergey Korablin") # required
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libc6 (>= 2.2.5)") # update as needed
set(CPACK_DEBIAN_PACKAGE_SECTION "utils") # update as needed
set(CPACK_DEBIAN_ARCHITECTURE ${CMAKE_SYSTEM_PROCESSOR})

# Include CPackDeb
include(CPackDeb)

