cmake_minimum_required(VERSION 3.14)

project(libserialport VERSION 0.1.1 LANGUAGES C)

set(SP_PACKAGE_VERSION_MAJOR ${PROJECT_VERSION_MAJOR})
set(SP_PACKAGE_VERSION_MINOR ${PROJECT_VERSION_MINOR})
set(SP_PACKAGE_VERSION_MICRO ${PROJECT_VERSION_PATCH})
set(SP_PACKAGE_VERSION_STRING "${PROJECT_VERSION}")

set(SP_LIB_VERSION_CURRENT 1)
set(SP_LIB_VERSION_REVISION 0)
set(SP_LIB_VERSION_AGE 1)
set(SP_LIB_VERSION_STRING "${SP_LIB_VERSION_CURRENT}:${SP_LIB_VERSION_REVISION}:${SP_LIB_VERSION_AGE}")
set(SP_LIB_VERSION_STRING_CLEAN "${SP_LIB_VERSION_CURRENT}.${SP_LIB_VERSION_REVISION}.${SP_LIB_VERSION_AGE}")

### configure config.h

include(CheckSymbolExists)
include(CheckIncludeFile)
include(CheckTypeSize)
include(CheckStructHasMember)

check_symbol_exists(clock_gettime "time.h" HAVE_CLOCK_GETTIME)

check_type_size("BOTHER" BOTHER_SIZE)
if (BOTHER_SIZE)
    set(HAVE_DECL_BOTHER 1)
endif()

check_include_file("dlfcn.h" HAVE_DLFCN_H)
check_include_file("sys/file.h" HAVE_SYS_FILE_H)
check_symbol_exists(flock "sys/file.h" HAVE_FLOCK)
check_symbol_exists(realpath "stdlib.h" HAVE_REALPATH)
check_include_file("inttypes.h" HAVE_INTTYPES_H)
check_include_file("memory.h" HAVE_MEMORY_H)
check_include_file("sys/stat.h" HAVE_SYS_STAT_H)
check_include_file("sys/types.h" HAVE_SYS_TYPES_H)
check_include_file("unistd.h" HAVE_UNISTD_H)
check_include_file("stdlib.h" HAVE_STDLIB_H)
check_include_file("strings.h" HAVE_STRINGS_H)
check_include_file("string.h" HAVE_STRING_H)

check_struct_has_member("struct termios" c_ispeed linux/termios.h HAVE_STRUCT_TERMIOS_C_ISPEED LANGUAGE C)
check_struct_has_member("struct termios" c_ospeed linux/termios.h HAVE_STRUCT_TERMIOS_C_OSPEED LANGUAGE C)

set(CMAKE_EXTRA_INCLUDE_FILES linux/termios.h)
check_type_size("struct termios2" TERMIOS2_SIZE)
if (TERMIOS2_SIZE)
    set(HAVE_STRUCT_TERMIOS2 1)
    check_struct_has_member("struct termios2" c_ispeed linux/termios.h HAVE_STRUCT_TERMIOS2_C_ISPEED LANGUAGE C)
    check_struct_has_member("struct termios2" c_ospeed linux/termios.h HAVE_STRUCT_TERMIOS2_C_OSPEED LANGUAGE C)
endif()

check_type_size("struct termiox" TERMIOX_SIZE)
if (TERMIOX_SIZE)
    set(HAVE_STRUCT_TERMIOX 1)
endif()

set(CMAKE_EXTRA_INCLUDE_FILES linux/serial.h)
check_type_size("struct serial_struct" SERIAL_STRUCT_SIZE)
if (SERIAL_STRUCT_SIZE)
    set(HAVE_STRUCT_SERIAL_STRUCT 1)
endif()

configure_file(cmake.config.h.in config.h)

### Library

include(GenerateExportHeader)

add_library(libserialport config.h serialport.c timing.c)
add_library(libserialport::libserialport ALIAS libserialport)

if (UNIX)
    target_sources(libserialport PRIVATE linux.c linux_termios.c)
elseif (WIN32)
    target_sources(libserialport PRIVATE windows.c)
endif()

generate_export_header(libserialport
    BASE_NAME libserialport
    EXPORT_MACRO_NAME SP_API
    NO_EXPORT_MACRO_NAME SP_PRIV
    EXPORT_FILE_NAME libserialport_export.h
    STATIC_DEFINE LIBSERIALPORT_STATIC_BUILD
)

target_include_directories(libserialport
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        $<INSTALL_INTERFACE:include>
)

# this define is needed for config.h inclusion
target_compile_definitions(libserialport PRIVATE LIBSERIALPORT_ATBUILD)

if (WIN32)
    target_link_libraries(libserialport PRIVATE setupapi)
endif()

set_target_properties(libserialport PROPERTIES PUBLIC_HEADER "${CMAKE_CURRENT_BINARY_DIR}/libserialport_export.h")
set_target_properties(libserialport PROPERTIES PUBLIC_HEADER libserialport.h)
set_target_properties(libserialport PROPERTIES VERSION ${SP_PACKAGE_VERSION_STRING})
set_target_properties(libserialport PROPERTIES POSITION_INDEPENDENT_CODE ON)
set_target_properties(libserialport PROPERTIES OUTPUT_NAME "serialport")

### Installation

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

install(
    TARGETS libserialport
    EXPORT ${CMAKE_PROJECT_NAME}Targets
)

install(
    FILES ${CMAKE_CURRENT_BINARY_DIR}/libserialport_export.h
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

install(
    EXPORT ${CMAKE_PROJECT_NAME}Targets
    FILE ${CMAKE_PROJECT_NAME}Targets.cmake
    NAMESPACE ${CMAKE_PROJECT_NAME}::
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${CMAKE_PROJECT_NAME}"
)

export(
    TARGETS ${CMAKE_PROJECT_NAME}
    NAMESPACE ${CMAKE_PROJECT_NAME}::
    FILE ${CMAKE_PROJECT_NAME}Targets.cmake
)
