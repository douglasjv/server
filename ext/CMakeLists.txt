# add_subdirectory(lua) # Handled globally
# add_subdirectory(mysql) # Handled globally
# add_subdirectory(zmq) # Handled globally

add_subdirectory(concurrentqueue)
add_subdirectory(detour)
add_subdirectory(recast)
add_subdirectory(sol)
add_subdirectory(spdlog)

# CPM Modules
CPMAddPackage(
    NAME argparse
    GITHUB_REPOSITORY p-ranav/argparse
    GIT_TAG f5ea927f6b3beae2ac924ffde27f570bb9a7d294
) # defines: argparse

CPMAddPackage(
    NAME efsw
    GITHUB_REPOSITORY SpartanJ/efsw
    GIT_TAG 74ca09bff89bc8de1f7b8bf3faaa6275ce23b4c5
    OPTIONS
        "VERBOSE OFF"
        "NO_ATOMICS OFF"
        "BUILD_SHARED_LIBS OFF"
        "BUILD_TEST_APP OFF"
        "EFSW_INSTALL OFF"
) # defines: efsw

CPMAddPackage(
    NAME asio
    VERSION 1.24.1
    GITHUB_REPOSITORY chriskohlhoff/asio
    GIT_TAG asio-1-24-0 # asio uses non-standard version tag, we must specify GIT_TAG
)

# ASIO doesn't use CMake, we have to configure it manually. Extra notes for using on Windows:
#
# If _WIN32_WINNT is not set, ASIO assumes _WIN32_WINNT=0x0501, i.e. Windows XP target, which is
# definitely not the platform which most users target.
if(asio_ADDED)
    add_library(asio INTERFACE)

    target_include_directories(asio SYSTEM INTERFACE ${asio_SOURCE_DIR}/asio/include)

    target_compile_definitions(asio INTERFACE ASIO_STANDALONE ASIO_NO_DEPRECATED)

    target_link_libraries(asio INTERFACE Threads::Threads)

    if(WIN32)
        # macro see @ https://stackoverflow.com/a/40217291/1746503
        macro(get_win32_winnt version)
            if(CMAKE_SYSTEM_VERSION)
                set(ver ${CMAKE_SYSTEM_VERSION})
                string(REGEX MATCH "^([0-9]+).([0-9])" ver ${ver})
                string(REGEX MATCH "^([0-9]+)" verMajor ${ver})

                # Check for Windows 10, b/c we'll need to convert to hex 'A'.
                if("${verMajor}" MATCHES "10")
                    set(verMajor "A")
                    string(REGEX REPLACE "^([0-9]+)" ${verMajor} ver ${ver})
                endif("${verMajor}" MATCHES "10")

                # Remove all remaining '.' characters.
                string(REPLACE "." "" ver ${ver})

                # Prepend each digit with a zero.
                string(REGEX REPLACE "([0-9A-Z])" "0\\1" ver ${ver})
                set(${version} "0x${ver}")
            endif()
        endmacro()

        if(NOT DEFINED _WIN32_WINNT)
            get_win32_winnt(ver)
            set(_WIN32_WINNT ${ver})
        endif()

        message(STATUS "Set _WIN32_WINNET=${_WIN32_WINNT}")

        target_compile_definitions(asio INTERFACE _WIN32_WINNT=${_WIN32_WINNT})
    endif()
endif()

set(EXTERNAL_LIBS
    concurrentqueue
    mariadbclient
    spdlog
    recast
    detour
    sol2_single
    argparse
    efsw
    asio
)

if(WIN32)
    # add wepoll for epoll support on windows
    add_subdirectory(wepoll)

    list(APPEND EXTERNAL_LIBS
        wepoll
    )

# backwards needs these to add to linker
elseif(APPLE)
    # MacOS has bfd built in clang
    list(APPEND EXTERNAL_LIBS
        dl
    )
elseif(UNIX)
    # Linux needs bfd as it's not native like MacOS
    list(APPEND EXTERNAL_LIBS
        bfd
        dl
    )
endif()

# copy external libs to parent scope
set(EXTERNAL_LIBS
    ${EXTERNAL_LIBS}
    PARENT_SCOPE
)
