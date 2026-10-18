# ===========================================================================
# libudfread — build definition.
#
# Reads the UDF filesystem, which is what an .iso holds when it is a DVD, a
# Blu-ray, or any image with a file past ISO 9660's 4 GiB ceiling (Windows
# installer images, most notably). libarchive covers ISO 9660 and has no UDF
# reader at all, so the two are complements: whichever one the image needs.
#
# Upstream builds with meson and ships no CMakeLists.txt, so the build is ours. It
# lives in the fork rather than in the application's tree because that is what makes
# the fork the complete answer to a source request: clone it and it builds, with no
# file from that repository needed. The source list, the feature checks, and the
# compile definitions below all mirror upstream's meson.build; if that file changes
# on a version bump, this is what to re-read.
#
# This is the one commit here upstream would never take. Leave it out of anything
# sent to VideoLAN.
#
# It is built static and linked into the application, as every dependency is, and
# the licence is what shapes how. libudfread is LGPL 2.1, which lets a closed
# program carry it on condition that whoever receives the program can relink it
# against their own build of the library (§6(a)). The relink kit each public
# release publishes is that: the program's own code as one object, every library
# it links, and the script that links them (the application's CMakeLists.txt,
# under "The relink kit").
#
# What that asks of this file:
#
#   * a static library, which the program links and the kit carries;
#   * nothing marked for export. UDFREAD_API_EXPORT, upstream's switch for a
#     shared library's exports, stays off: in a static library it would make the
#     program export the API instead;
#   * the licence text has to reach a recipient of the binary. It does, as
#     `licenses/License-libudfread.txt` in the application tree, shipped with
#     every other component's notice — nothing is installed from here.
#
# We also modify the library — src/udfread.c selects the partition the Logical
# Volume Descriptor names, the reasoning being where that change is — which makes
# what we ship a modified libudfread, and a recipient is owed that source. This
# repository is it: the fork carries the change as commits and this file as its
# build, so "here is our fork, at this commit" is the whole of the answer.
# ===========================================================================
cmake_minimum_required(VERSION 3.21)
project(libudfread LANGUAGES C)

include(CheckIncludeFile)
include(CheckSymbolExists)

# Set by the application's dependency build too, and repeated for a standalone
# clone; find_package is idempotent, so the parent's result simply stands.
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

set(NCOMM_UDFREAD_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
set(NCOMM_UDFREAD_GEN "${CMAKE_CURRENT_BINARY_DIR}")

if(NOT EXISTS "${NCOMM_UDFREAD_DIR}/src/udfread.c")
    message(FATAL_ERROR
        "3rdparty/libudfread is empty — the submodule has not been checked out.\n"
        "Run:  git submodule update --init --recursive")
endif()

# --- config.h ---------------------------------------------------------------
# The sources compile against a generated config.h (guarded by HAVE_CONFIG_H)
# and read HAVE_UNISTD_H / HAVE_FCNTL_H from it to decide whether to include
# <unistd.h> and <fcntl.h> at all. Without it they call open() and pread() with
# no declaration in scope, which modern C compilers reject outright.
#
# _POSIX_C_SOURCE has to be set for the checks too, not just the build: it is
# what exposes pread() in the headers, so checking without it would come back
# negative on a platform that has it.
if(NOT WIN32)
    list(APPEND CMAKE_REQUIRED_DEFINITIONS -D_POSIX_C_SOURCE=200809L)
endif()
check_include_file(unistd.h HAVE_UNISTD_H)
check_include_file(fcntl.h  HAVE_FCNTL_H)
if(NOT WIN32)
    check_symbol_exists(pread "unistd.h" NCOMM_UDFREAD_HAVE_PREAD)
    if(NOT NCOMM_UDFREAD_HAVE_PREAD)
        message(FATAL_ERROR "libudfread requires pread(), which this platform does not provide.")
    endif()
    if(CMAKE_USE_PTHREADS_INIT)
        set(HAVE_PTHREAD_H 1)
    endif()
endif()
set(HAVE_UDFREAD_VERSION_H 1)  # we generate it, just below

configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.h.in"
    "${NCOMM_UDFREAD_GEN}/config.h")

# --- udfread-version.h ------------------------------------------------------
# Generated from upstream's template. These must match the tag the submodule is
# pinned to (1.2.0) or the library will misreport its own version.
set(UDFREAD_VERSION_MAJOR 1)
set(UDFREAD_VERSION_MINOR 2)
set(UDFREAD_VERSION_MICRO 0)
configure_file(
    "${NCOMM_UDFREAD_DIR}/src/udfread-version.h.in"
    "${NCOMM_UDFREAD_GEN}/udfread-version.h"
    @ONLY)

# --- The library ------------------------------------------------------------
# static, as every dependency of the application is. The parent CMakeLists sets
# BUILD_SHARED_LIBS off for all of them; the keyword says the same for a standalone
# clone of the fork. The licence reason is at the top of this file.
add_library(udfread STATIC
    "${NCOMM_UDFREAD_DIR}/src/udfread.c"
    "${NCOMM_UDFREAD_DIR}/src/ecma167.c"
    "${NCOMM_UDFREAD_DIR}/src/default_blockinput.c"
    "${NCOMM_UDFREAD_DIR}/src/udfread-version.c"
)

# Hidden, as upstream's meson builds it (gnu_symbol_visibility: 'hidden'): none of
# the library's symbols is exported from the program it is linked into.
set_target_properties(udfread PROPERTIES C_VISIBILITY_PRESET hidden)

target_include_directories(udfread
    PUBLIC  "${NCOMM_UDFREAD_DIR}/src"
    PRIVATE "${NCOMM_UDFREAD_GEN}")   # the generated config.h and version header

# HAVE_CONFIG_H and _ISOC99_SOURCE are unconditional in upstream's meson;
# _POSIX_C_SOURCE is what makes pread() visible off Windows. UDFREAD_API_EXPORT is
# left off (the note at the top), so UDF_PUBLIC is nothing: on Windows it would be
# dllexport, and the program would export the API with an import library of its
# own written beside the executable.
target_compile_definitions(udfread PRIVATE HAVE_CONFIG_H=1 _ISOC99_SOURCE)
if(NOT WIN32)
    target_compile_definitions(udfread PRIVATE _POSIX_C_SOURCE=200809L)
endif()
if(WIN32)
    # udfread.h declares udfread_file_read() returning ssize_t, which Windows has in no
    # standard header — so the library and every consumer of the header need it defined.
    # PUBLIC (not PRIVATE) because it is the PUBLIC header that needs it; the installed
    # prefix carries the same define on ncomm::udfread (see ncomm-deps.cmake), since the
    # app consumes the prefix and never this target. A build definition rather than a
    # source change, because it is the header's consumers that need it, not the library.
    target_compile_definitions(udfread PUBLIC ssize_t=ptrdiff_t)
endif()

target_link_libraries(udfread PRIVATE Threads::Threads)

# Install into the bootstrap prefix. Upstream builds with meson, so its CMake —
# and therefore its install rules — are ours. A static library is one file:
# libudfread.a, or udfread.lib on Windows.
install(TARGETS udfread ARCHIVE DESTINATION lib)
install(FILES "${NCOMM_UDFREAD_DIR}/src/udfread.h"
              "${NCOMM_UDFREAD_DIR}/src/blockinput.h"
              "${NCOMM_UDFREAD_GEN}/udfread-version.h"
        DESTINATION include/udfread)

# no licence is installed from here. The LGPL text still has to travel with the
# binary, but it travels the way every other component's does: as a file in the
# application's own `licenses/` directory, which the app's build ships whole. A
# second route for one component meant one component's notice could go stale or
# go missing on its own, which is exactly the failure this directory exists to
# prevent. Do not add an install() rule here for copying.
