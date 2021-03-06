project ("ev3devkit")

cmake_minimum_required (VERSION 2.6)

get_directory_property(PARENT_DIR PARENT_DIRECTORY)

if (NOT PARENT_DIR)
    option (EV3DEVKIT_DESKTOP "Build desktop library")
    option (EV3DEVKIT_BUILD_DEMO "Build the demo programs" Yes)
    option (BUILD_SHARED_LIBS "Build shared library instead of static")
endif (NOT PARENT_DIR)

list (APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

set (DATADIR "${CMAKE_INSTALL_PREFIX}/share")

# Use CMAKE_PROJECT_NAME if we are building a statically linked library.
# This way, if there is a parent project, it uses that name instead.
if (BUILD_SHARED_LIBS)
    set (PKGDATADIR "${DATADIR}/${PROJECT_NAME}")
else (BUILD_SHARED_LIBS)
    set (PKGDATADIR "${DATADIR}/${CMAKE_PROJECT_NAME}")
endif (BUILD_SHARED_LIBS)
set (DEVICES_DEMO_EXEC_NAME "ev3devKit-devices-demo")
set (UI_DEMO_EXEC_NAME "ev3devKit-ui-demo")
set (VERSION "0.1")

set (EV3DEVKIT_PACKAGE ${PROJECT_NAME}-${VERSION})
set (EV3DEVKIT_DESKTOP_PACKAGE ${PROJECT_NAME}-desktop-${VERSION})
set (GIR_NAME "EV3devKit")
set (LIBRARY_NAME "ev3devKit")
set (DESKTOP_LIBRARY_NAME "ev3devKit-desktop")
set (TARGET_GLIB "2.40")

if (PARENT_DIR)
    # If this is included by a parent project, export some variables back to the parent
    set (${PROJECT_NAME}_PACKAGE ${EV3DEVKIT_PACKAGE} PARENT_SCOPE)
    set (${PROJECT_NAME}_VAPI ${EV3DEVKIT_PACKAGE}.vapi PARENT_SCOPE)
    set (${PROJECT_NAME}_DESKTOP_PACKAGE ${EV3DEVKIT_DESKTOP_PACKAGE} PARENT_SCOPE)
    set (${PROJECT_NAME}_DESKTOP_VAPI ${EV3DEVKIT_DESKTOP_PACKAGE}.vapi PARENT_SCOPE)
endif (PARENT_DIR)

add_subdirectory (data)

# Comment this out to enable C compiler warnings
add_definitions (-w)

set (COMMON_PACKAGES
    glib-2.0
    gobject-2.0
    gmodule-2.0
    gio-2.0
    gio-unix-2.0
    gee-0.8
    gudev-1.0
    curses
    posix
    linux
)
set (COMMON_MODULES
    glib-2.0
    gobject-2.0
    gmodule-2.0
    gio-2.0
    gio-unix-2.0
    gee-0.8
    gudev-1.0
    ncurses
)
set (DESKTOP_MODULES gtk+-3.0)
set (DESKTOP_PACKAGES gtk+-3.0)

configure_file (src/Version.vala.in ${CMAKE_CURRENT_BINARY_DIR}/Version.vala)

if (EV3DEVKIT_DESKTOP)
    set (UI_DEMO_SOURCE_FILES
        demo/UIDemoWindow.vala
        demo/ui_desktop_demo.vala
    )
else (EV3DEVKIT_DESKTOP)
    set (UI_DEMO_SOURCE_FILES
        demo/UIDemoWindow.vala
        demo/ui_demo.vala
    )
endif (EV3DEVKIT_DESKTOP)

set (DEVICES_DEMO_SOURCE_FILES
    demo/devices_demo.vala
)

set (LIBRARY_SOURCE_FILES
    src/devices/DCMotor.vala
    src/devices/Device.vala
    src/devices/DeviceManager.vala
    src/devices/Input.vala
    src/devices/LED.vala
    src/devices/MotorPolarity.vala
    src/devices/MotorStateFlags.vala
    src/devices/Port.vala
    src/devices/PowerSupply.vala
    src/devices/Sensor.vala
    src/devices/ServoMotor.vala
    src/devices/TachoMotor.vala
    src/ui/Box.vala
    src/ui/Button.vala
    src/ui/CheckboxMenuItem.vala
    src/ui/CheckButton.vala
    src/ui/Container.vala
    src/ui/Dialog.vala
    src/ui/Grid.vala
    src/ui/Icon.vala
    src/ui/Label.vala
    src/ui/Menu.vala
    src/ui/MenuItem.vala
    src/ui/MessageDialog.vala
    src/ui/Notebook.vala
    src/ui/NotebookTab.vala
    src/ui/OnScreenKeyboard.vala
    src/ui/RadioMenuItem.vala
    src/ui/Rectangle.vala
    src/ui/Screen.vala
    src/ui/Scroll.vala
    src/ui/Spacer.vala
    src/ui/StatusBar.vala
    src/ui/StatusBarItem.vala
    src/ui/TabButton.vala
    src/ui/TextEntry.vala
    src/ui/Widget.vala
    src/ui/Window.vala
    src/ConsoleApp.vala
    ${CMAKE_CURRENT_BINARY_DIR}/Version.vala
)

set (DESKTOP_LIBRARY_SOURCE_FILES
    src/desktop/GtkApp.vala
    src/desktop/GtkFramebuffer.vala
    src/desktop/GtkScreen.vala
)

find_package (PkgConfig REQUIRED)

pkg_check_modules (DEPS REQUIRED ${COMMON_MODULES})
add_definitions (${DEPS_CFLAGS})
link_libraries (${DEPS_LIBRARIES})
link_directories (${DEPS_LIBRARY_DIRS})

if (EV3DEVKIT_DESKTOP)
    pkg_check_modules (DESKTOP_DEPS REQUIRED ${DESKTOP_MODULES})
    add_definitions (${DESKTOP_DEPS_CFLAGS})
    link_libraries (${DESKTOP_DEPS_LIBRARIES})
    link_directories (${DESKTOP_DEPS_LIBRARY_DIRS})
endif (EV3DEVKIT_DESKTOP)

link_libraries (grx20 m)

find_package (Vala REQUIRED)
include (ValaVersion)
ensure_vala_version ("0.24" MINIMUM)
include (ValaPrecompile)

vala_precompile (LIBRARY_VALA_C ${LIBRARY_NAME}
LIBRARY
    ${LIBRARY_SOURCE_FILES}
PACKAGES
    ${COMMON_PACKAGES}
CUSTOM_VAPIS
    bindings/*.vapi
OPTIONS
    --target-glib=${TARGET_GLIB}
    --thread
GENERATE_VAPI
    ${EV3DEVKIT_PACKAGE}.buggy
GENERATE_HEADER
    ${EV3DEVKIT_PACKAGE}
GENERATE_GIR TYPELIB
    ${GIR_NAME}-${VERSION}
)

add_library (${LIBRARY_NAME} ${LIBRARY_VALA_C})
add_dependencies(${LIBRARY_NAME} ev3devkit_data)
# only install static library if this is not a subproject
if (BUILD_SHARED_LIBS OR NOT PARENT_DIR)
    install (TARGETS ${LIBRARY_NAME} LIBRARY DESTINATION lib ARCHIVE DESTINATION lib)
endif (BUILD_SHARED_LIBS OR NOT PARENT_DIR)

# There is a bug in valac that causes an unowned compact class to lose
# the "unowned" qualifier when generating a vapi. So, we are calling a
# script we wrote to fix it.
add_custom_command (OUTPUT
    ${CMAKE_CURRENT_BINARY_DIR}/${EV3DEVKIT_PACKAGE}.vapi
COMMAND
    ${CMAKE_CURRENT_SOURCE_DIR}/fix-vapi.sh
    ${CMAKE_CURRENT_BINARY_DIR}/${EV3DEVKIT_PACKAGE}.buggy.vapi
    ${CMAKE_CURRENT_BINARY_DIR}/${EV3DEVKIT_PACKAGE}.vapi
DEPENDS
    ${CMAKE_CURRENT_SOURCE_DIR}/fix-vapi.sh
    ${CMAKE_CURRENT_BINARY_DIR}/${EV3DEVKIT_PACKAGE}.buggy.vapi
COMMENT
    Fixing ${CMAKE_CURRENT_BINARY_DIR}/${EV3DEVKIT_PACKAGE}.buggy.vapi
)

# Workaround to make CMake 2.8 generate ev3devKit.vapi
# apparently add_custom_command is ignored if nothing depends on its output.
set (FIX_VAPI_DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/${EV3DEVKIT_PACKAGE}.vapi)
add_custom_target (fix-vapi DEPENDS ${FIX_VAPI_DEPENDS})
add_dependencies (${LIBRARY_NAME} fix-vapi)

if (EV3DEVKIT_DESKTOP)
    vala_precompile (DESKTOP_LIBRARY_VALA_C ${DESKTOP_LIBRARY_NAME}
    LIBRARY
        ${DESKTOP_LIBRARY_SOURCE_FILES}
    PACKAGES
        ${COMMON_PACKAGES}
        ${DESKTOP_PACKAGES}
    CUSTOM_VAPIS
        bindings/*.vapi
        ${CMAKE_CURRENT_BINARY_DIR}/${EV3DEVKIT_PACKAGE}.vapi
    OPTIONS
        --target-glib=${TARGET_GLIB}
        --thread
    GENERATE_VAPI
        ${EV3DEVKIT_DESKTOP_PACKAGE}
    GENERATE_HEADER
        ${DESKTOP_LIBRARY_NAME}
    GENERATE_GIR
       ${GIR_NAME}Desktop-buggy-${VERSION}
    )

    # Work around bug in valac where .gir is not correctly generated with nested
    # namespaces. The EV3devKit .gir is only correctly generated if we leave out
    # the gir_namespace and gir_version CCode attributes. However, this causes
    # the EV3DevKitDesktop .gir to not add an include line for EV3devKit. So,
    # this workaround just adds that line.

    add_custom_command (
        OUTPUT
            "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}Desktop-${VERSION}.gir"
        COMMAND
            "${CMAKE_CURRENT_SOURCE_DIR}/fix-desktop-gir.sh"
        ARGS
            ${GIR_NAME}
            ${VERSION}
            "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}Desktop-buggy-${VERSION}.gir"
            "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}Desktop-${VERSION}.gir"
        DEPENDS
            "${CMAKE_CURRENT_SOURCE_DIR}/fix-desktop-gir.sh"
            "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}Desktop-buggy-${VERSION}.gir"
        COMMENT
            "Fixing ${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}Desktop-buggy-${VERSION}.gir"
    )

    add_custom_command (
        OUTPUT
            "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}Desktop-${VERSION}.typelib"
        COMMAND
            ${G_IR_COMPILER_EXECUTABLE}
        ARGS
            "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}Desktop-${VERSION}.gir"
            "--shared-library=lib${DESKTOP_LIBRARY_NAME}"
            "--output=${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}Desktop-${VERSION}.typelib"
            "--includedir=${CMAKE_CURRENT_BINARY_DIR}"
        DEPENDS
            "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}Desktop-${VERSION}.gir"
        COMMENT
            "Genterating typelib."
    )

    add_custom_target ("${DESKTOP_LIBRARY_NAME}-typelib"
        ALL
        DEPENDS
            "${CMAKE_CURRENT_BINARY_DIR}/${GIR_NAME}Desktop-${VERSION}.typelib"
    )

    # end of workaround

    add_library (${DESKTOP_LIBRARY_NAME} ${DESKTOP_LIBRARY_VALA_C})
    target_link_libraries (${DESKTOP_LIBRARY_NAME} ${LIBRARY_NAME})
    add_dependencies (${DESKTOP_LIBRARY_NAME} ${LIBRARY_NAME})
    # only install static library if this is not a subproject
    if (BUILD_SHARED_LIBS OR NOT PARENT_DIR)
        install (TARGETS ${DESKTOP_LIBRARY_NAME} LIBRARY DESTINATION lib ARCHIVE DESTINATION lib)
    endif (BUILD_SHARED_LIBS OR NOT PARENT_DIR)
    configure_file (
        ${CMAKE_CURRENT_SOURCE_DIR}/src/desktop/main_window.glade
        ${CMAKE_BINARY_DIR}/main_window.glade COPYONLY
    )
endif (EV3DEVKIT_DESKTOP)

if (EV3DEVKIT_BUILD_DEMO)
    set (EXEC_PACKAGES ${COMMON_PACKAGES} ${EV3DEVKIT_PACKAGE})
    if (EV3DEVKIT_DESKTOP)
        list (APPEND EXEC_PACKAGES ${DESKTOP_PACKAGES} ${EV3DEVKIT_DESKTOP_PACKAGE})
    endif (EV3DEVKIT_DESKTOP)

    set (EXEC_LIBRARY_DEPENDS ${LIBRARY_NAME})
    if (EV3DEVKIT_DESKTOP)
        list (APPEND EXEC_LIBRARY_DEPENDS ${DESKTOP_LIBRARY_NAME})
    endif (EV3DEVKIT_DESKTOP)

    # Device driver demo program

    vala_precompile(DEVICES_DEMO_EXEC_VALA_C ${DEVICES_DEMO_EXEC_NAME}
        ${DEVICES_DEMO_SOURCE_FILES}
    PACKAGES
        ${EXEC_PACKAGES}
    CUSTOM_VAPIS
        bindings/*.vapi
    OPTIONS
        --target-glib=${TARGET_GLIB}
        --thread
        --vapidir=${CMAKE_CURRENT_BINARY_DIR}
    )

    add_executable (${DEVICES_DEMO_EXEC_NAME} ${DEVICES_DEMO_EXEC_VALA_C})
    target_link_libraries (${DEVICES_DEMO_EXEC_NAME} ${EXEC_LIBRARY_DEPENDS})

    add_custom_target (run-${DEVICES_DEMO_EXEC_NAME}
        COMMAND ${DEVICES_DEMO_EXEC_NAME}
        DEPENDS ${DEVICES_DEMO_EXEC_NAME}
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )

    install (TARGETS ${DEVICES_DEMO_EXEC_NAME} RUNTIME DESTINATION bin)

    # User interface demo program

    vala_precompile(UI_DEMO_EXEC_VALA_C ${UI_DEMO_EXEC_NAME}
        ${UI_DEMO_SOURCE_FILES}
    PACKAGES
        ${EXEC_PACKAGES}
    CUSTOM_VAPIS
        bindings/*.vapi
    OPTIONS
        --target-glib=${TARGET_GLIB}
        --thread
        --vapidir=${CMAKE_CURRENT_BINARY_DIR}
    )

    set (EXEC_LIBRARY_DEPENDS ${LIBRARY_NAME})
    if (EV3DEVKIT_DESKTOP)
        list (APPEND EXEC_LIBRARY_DEPENDS ${DESKTOP_LIBRARY_NAME})
    endif (EV3DEVKIT_DESKTOP)

    add_executable (${UI_DEMO_EXEC_NAME} ${UI_DEMO_EXEC_VALA_C})
    target_link_libraries (${UI_DEMO_EXEC_NAME} ${EXEC_LIBRARY_DEPENDS})

    add_custom_target (run-${UI_DEMO_EXEC_NAME}
        COMMAND ${UI_DEMO_EXEC_NAME}
        DEPENDS ${UI_DEMO_EXEC_NAME}
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )

    install (TARGETS ${UI_DEMO_EXEC_NAME} RUNTIME DESTINATION bin)

    # python user interface demo (requires shared library, i.e. BUILD_SHARED_LIBS is true)

    add_custom_target (run-python-ui-desktop-demo
        COMMAND
            LD_LIBRARY_PATH=${CMAKE_CURRENT_BINARY_DIR}
            GI_TYPELIB_PATH=${CMAKE_CURRENT_BINARY_DIR}
            ${CMAKE_CURRENT_SOURCE_DIR}/demo/python/ui_desktop_demo.py
        DEPENDS
            ${LIBRARY_NAME}
            ${DESKTOP_LIBRARY_NAME}
            ${CMAKE_CURRENT_SOURCE_DIR}/demo/python/ui_demo_window.py
            ${CMAKE_CURRENT_SOURCE_DIR}/demo/python/ui_desktop_demo.py
        WORKING_DIRECTORY
            ${CMAKE_BINARY_DIR}
    )
endif (EV3DEVKIT_BUILD_DEMO)

find_package (Valadoc)
if (VALADOC_FOUND)
    include (ValadocGenerate)
    generate_valadoc (
        ${LIBRARY_SOURCE_FILES}
        ${DESKTOP_LIBRARY_SOURCE_FILES}
        bindings/*.vapi
    PACKAGE_NAME
        ${PROJECT_NAME}
    PACKAGE_VERSION
        ${VERSION}
    OPTIONS
        --target-glib=${TARGET_GLIB}
    PACKAGES
        ${COMMON_PACKAGES}
        ${DESKTOP_PACKAGES}
    )
endif (VALADOC_FOUND)
