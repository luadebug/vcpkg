cmake_minimum_required(VERSION 3.5)
project(cmake-user LANGUAGES C)

message(STATUS "CMAKE_COMMAND: ${CMAKE_COMMAND}")
set(CHECK_CMAKE_VERSION "NOTFOUND" CACHE STRING "Version of CMake expected to be found")
if(NOT CHECK_CMAKE_VERSION)
    message(WARNING "CMake version check: skipped (actual: ${CMAKE_VERSION})")
elseif(NOT CHECK_CMAKE_VERSION VERSION_EQUAL CMAKE_VERSION)
    message(SEND_ERROR "CMake version check: failed (actual: ${CMAKE_VERSION} expected: ${CHECK_CMAKE_VERSION})")
else()
    message(STATUS "CMake version check: success (actual: ${CMAKE_VERSION})")
endif()

# add_library overload
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/lib.c"  "int lib_unused() { return 1; }")
add_library(lib "${CMAKE_CURRENT_BINARY_DIR}/lib.c")

# add_executable overload
file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/exe.c"  "int main() { return 0; }")
add_executable(exe "${CMAKE_CURRENT_BINARY_DIR}/exe.c")

# install overload
set(X_VCPKG_APPLOCAL_DEPS_INSTALL 1)
install(TARGETS exe lib
  RUNTIME DESTINATION bin
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
)

# find_package overload and wrapper
set(FIND_PACKAGES "" CACHE STRING "List of packages to be found and used")
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}")
foreach(package ${FIND_PACKAGES})
    set(extra_args "")
    if(package STREQUAL "ICU")
        set(extra_args COMPONENTS uc)
    endif()
    string(TOUPPER "${package}" package_upper)
    set(CMAKE_FIND_DEBUG_MODE ON)
    find_package("${package}" ${extra_args} QUIET)
    set(CMAKE_FIND_DEBUG_MODE OFF)
    if(NOT ${package}_FOUND AND NOT ${package_upper}_FOUND)
        message(SEND_ERROR "find_package(${package} ${extra_args}) check: failed")
        continue()
    endif()
    # REQUIRED changes the behaviour find_package_handle_standard_args.
    find_package("${package}" ${extra_args} REQUIRED)
    message(STATUS "find_package(${package} ${extra_args}) check: success")

    set(libraries_var "")
    if(DEFINED ${package}_LIBRARIES)
        set(libraries_var "${package}_LIBRARIES")
    elseif(DEFINED ${package_upper}_LIBRARIES)
        set(libraries_var "${package_upper}_LIBRARIES")
    elseif(DEFINED ${package}_LIBRARY)
        set(libraries_var "${package}_LIBRARY")
    elseif(DEFINED ${package_upper}_LIBRARY)
        set(libraries_var "${package_upper}_LIBRARY")
    else()
        message(STATUS "${package}_LIBRARY/IES: undefined")
        continue()
    endif()
    set(libraries "${${libraries_var}}")
    message(STATUS "${libraries_var}: ${libraries}")

    if(package STREQUAL "Intl" AND NOT Intl_LIBRARY)
        continue() # using libintl.h from C runtime library
    endif()
    target_link_libraries(exe PRIVATE ${libraries})

    set(last_keyword "")
    foreach(item IN LISTS libraries)
        if(item STREQUAL "optimized" OR item STREQUAL "debug")
            set(last_keyword "${item}")
            continue()
        endif()
        string(FIND "${item}" "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/lib/" starts_with_release)
        string(FIND "${item}" "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/debug/lib/" starts_with_debug)
        if(starts_with_release EQUAL "0")
            if(last_keyword STREQUAL "optimized")
                # okay
            elseif(last_keyword STREQUAL "debug")
                message(SEND_ERROR "Release lib for 'debug' keyword: ${item}")
            elseif(CMAKE_BUILD_TYPE STREQUAL "Debug")
                message(SEND_ERROR "Release lib for 'Debug' build: ${item}")
            endif()
        elseif(starts_with_debug EQUAL "0")
            if(last_keyword STREQUAL "debug")
                # okay
            elseif(last_keyword STREQUAL "optimized")
                message(SEND_ERROR "Debug lib for 'optimized' keyword: ${item}")
            elseif(CMAKE_BUILD_TYPE STREQUAL "Release")
                message(SEND_ERROR "Debug lib for 'Release' build: ${item}")
            endif()
        endif()
        set(last_keyword "")
        continue()
    endforeach()

    set(target "${package}::${package}")
    string(REPLACE "SQLite3::" "SQLite::" target "${target}")
    if(TARGET "${target}")
        get_target_property(type "${target}" TYPE)
        message(STATUS "${target}: ${type}")
        if(type MATCHES "LIBRARY" AND NOT type MATCHES "INTERFACE")
            get_target_property(configurations "${target}" IMPORTED_CONFIGURATIONS)
            message(STATUS "IMPORTED_CONFIGURATIONS: ${configurations}")
            if(configurations)
                string(TOLOWER "${configurations}" configurations)
                if("release" IN_LIST configurations)
                    set(property IMPORTED_IMPLIB_RELEASE)
                    get_target_property(location "${target}" "${property}")
                    if(NOT location)
                        set(property IMPORTED_LOCATION_RELEASE)
                        get_target_property(location "${target}" "${property}")
                    endif()
                    message(STATUS "${property}: ${location}")
                    string(FIND "${location}" "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/lib" index)
                    if (NOT index EQUAL "0")
                        message(SEND_ERROR "Release lib is in wrong location.")
                    endif()
                elseif(NOT CHECK_BUILD_TYPE OR CHECK_BUILD_TYPE STREQUAL "release")
                    message(SEND_ERROR "Release configuration is missing.")
                endif()
                if("debug" IN_LIST configurations)
                    set(property IMPORTED_IMPLIB_DEBUG)
                    get_target_property(location "${target}" "${property}")
                    if(NOT location)
                        set(property IMPORTED_LOCATION_DEBUG)
                        get_target_property(location "${target}" "${property}")
                    endif()
                    message(STATUS "${property}: ${location}")
                    string(FIND "${location}" "${VCPKG_INSTALLED_DIR}/${VCPKG_TARGET_TRIPLET}/debug/lib" index)
                    if (NOT index EQUAL "0")
                        message(SEND_ERROR "Debug lib is in wrong location.")
                    endif()
                elseif(NOT CHECK_BUILD_TYPE OR CHECK_BUILD_TYPE STREQUAL "debug")
                    message(SEND_ERROR "Debug configuration is missing.")
                endif()
            endif()
        endif()
    else()
        get_directory_property(imported_targets IMPORTED_TARGETS)
        if(imported_targets)
            message(STATUS "Imported targets: ${imported_targets}")
        endif()
    endif()
endforeach()
