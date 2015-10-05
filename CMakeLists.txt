cmake_minimum_required(VERSION 3.0.0)

# Extract project version from configure.ac
file(READ configure.ac CONFIGURE_AC_CONTENTS)
string(REGEX MATCH "AC_INIT\\(\\[libogg\\],\\[([0-9]*).([0-9]*).([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
set(PROJECT_VERSION_MAJOR ${CMAKE_MATCH_1})
set(PROJECT_VERSION_MINOR ${CMAKE_MATCH_2})
set(PROJECT_VERSION_PATCH ${CMAKE_MATCH_3})
set(PROJECT_VERSION ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH})

project(ogg VERSION ${PROJECT_VERSION})

set(CMAKE_DEBUG_POSTFIX "d")

# Required modules
include(GNUInstallDirs)
include(CheckIncludeFiles)
include(CMakePackageConfigHelpers)

# Build options
option(BUILD_SHARED_LIBS "Build shared library" OFF)
if (MSVC)
    option(MSVC_LINK_STATIC_RUNTIME "Use static msvc runtime" OFF)
endif()

if (APPLE)
    option(BUILD_FRAMEWORK "Build Framework bundle for OSX" OFF)
endif()

# statically link msvc runtime
if (MSVC AND MSVC_LINK_STATIC_RUNTIME)
	foreach(flag_var
		CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
		CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO)

		if(${flag_var} MATCHES "/MD")
			string(REGEX REPLACE "/MD" "/MT" ${flag_var} "${${flag_var}}")
		endif(${flag_var} MATCHES "/MD")
	endforeach(flag_var)
endif()

# Helper function to get version-info
function(get_version_info result current_var_name age_var_name revision_var_name)
    string(REGEX MATCH "${current_var_name}=([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
    set(VERSION_INFO_CURRENT ${CMAKE_MATCH_1})

    string(REGEX MATCH "${age_var_name}=([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
    set(VERSION_INFO_AGE ${CMAKE_MATCH_1})

    string(REGEX MATCH "${revision_var_name}=([0-9]*)" DUMMY ${CONFIGURE_AC_CONTENTS})
    set(VERSION_INFO_REVISION ${CMAKE_MATCH_1})

    math(EXPR VERSION_INFO_CURRENT_MINUS_AGE "${VERSION_INFO_CURRENT} - ${VERSION_INFO_AGE}")

    set(${result} "${VERSION_INFO_CURRENT_MINUS_AGE}.${VERSION_INFO_AGE}.${VERSION_INFO_REVISION}" PARENT_SCOPE)
endfunction()

# Helper function to configure pkg-config files
function(configure_pkg_config_file pkg_config_file_in)
    set(prefix ${CMAKE_INSTALL_PREFIX})
    set(exec_prefix ${CMAKE_INSTALL_FULL_BINDIR})
    set(libdir ${CMAKE_INSTALL_FULL_LIBDIR})
    set(includedir ${CMAKE_INSTALL_FULL_INCLUDEDIR})
    set(VERSION ${PROJECT_VERSION})
    string(REPLACE ".in" "" pkg_config_file ${pkg_config_file_in})
    configure_file(${pkg_config_file_in} ${CMAKE_CURRENT_SOURCE_DIR}/${pkg_config_file} @ONLY)
endfunction()

message(STATUS "Configuring ${PROJECT_NAME} ${PROJECT_VERSION}")

# Configure config_type.h
check_include_files(inttypes.h INCLUDE_INTTYPES_H)
check_include_files(stdint.h INCLUDE_STDINT_H)
check_include_files(sys/types.h INCLUDE_SYS_TYPES_H)

set(SIZE16 int16_t)
set(USIZE16 uint16_t)
set(SIZE32 int32_t)
set(USIZE32 uint32_t)
set(SIZE64 int64_t)

configure_file(include/ogg/config_types.h.in ${CMAKE_CURRENT_BINARY_DIR}/include/ogg/config_types.h @ONLY)

set(OGG_HEADERS
    include/ogg/ogg.h
    include/ogg/os_types.h
)

set(OGG_SOURCES
    src/bitwise.c
    src/framing.c
)

if(MSVC)
    list(APPEND OGG_SOURCES win32/ogg.def)
endif()

if (BUILD_FRAMEWORK AND NOT BUILD_SHARED_LIBS)
    message(STATUS "Building framework... so enabling shared library build")
    set(BUILD_SHARED_LIBS TRUE)
endif()

include_directories(include)
list(APPEND private_include_dirs
    "${CMAKE_CURRENT_SOURCE_DIR}/include"
    "${CMAKE_CURRENT_BINARY_DIR}/include")

get_version_info(OGG_VERSION_INFO "LIB_CURRENT" "LIB_AGE" "LIB_REVISION")

add_library(libogg ${OGG_HEADERS} ${OGG_SOURCES})
set_target_properties(libogg PROPERTIES
    SOVERSION ${OGG_VERSION_INFO}
    INCLUDE_DIRECTORIES "${private_include_dirs}"
    INTERFACE_INCLUDE_DIRECTORIES "$<BUILD_INTERFACE:${private_include_dirs}>;$<INSTALL_INTERFACE:include>"
    IMPORT_SUFFIX "_imp${CMAKE_IMPORT_LIBRARY_SUFFIX}"
    PREFIX ""
    IMPORT_PREFIX ""
    )

if(BUILD_FRAMEWORK)
    set_target_properties(libogg PROPERTIES
        FRAMEWORK TRUE
        FRAMEWORK_VERSION ${PROJECT_VERSION}
        MACOSX_FRAMEWORK_IDENTIFIER org.xiph.ogg
        MACOSX_FRAMEWORK_SHORT_VERSION_STRING ${PROJECT_VERSION}
        MACOSX_FRAMEWORK_BUNDLE_VERSION ${PROJECT_VERSION}
        XCODE_ATTRIBUTE_INSTALL_PATH "@rpath"
        PUBLIC_HEADER "${OGG_HEADERS}")
endif()

configure_pkg_config_file(ogg.pc.in)

write_basic_package_version_file("${PROJECT_NAME}ConfigVersion.cmake" COMPATIBILITY SameMajorVersion)

install(FILES
    include/ogg/ogg.h
    include/ogg/os_types.h
    ${CMAKE_CURRENT_BINARY_DIR}/include/ogg/config_types.h
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/ogg)

install(TARGETS libogg
    EXPORT ${PROJECT_NAME}Config
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
    FRAMEWORK DESTINATION lib)

configure_file(ogg.pc.in ${CMAKE_CURRENT_BINARY_DIR}/ogg.pc COPYONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/ogg.pc
    DESTINATION lib/pkgconfig)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
	DESTINATION lib/cmake)

install(EXPORT ${PROJECT_NAME}Config
	DESTINATION lib/cmake)
