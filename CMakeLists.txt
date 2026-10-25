cmake_minimum_required(VERSION 3.21)
project(halide-llvm LANGUAGES C CXX)

# =============================================================================
# 1. Retrieve the Ref (Source of Truth)
# =============================================================================
set(HALIDE_LLVM_REF "$ENV{HALIDE_LLVM_REF}")
if (NOT HALIDE_LLVM_REF)
    message(FATAL_ERROR
        "HALIDE_LLVM_REF environment variable is required.\n"
        "Examples: 'llvmorg-21.1.8', 'main', or a commit SHA"
    )
endif ()

# =============================================================================
# 2. Calculate the Source Path
# =============================================================================
# Must match Python's sanitize_ref_for_path logic: replace invalid chars with _
string(REGEX REPLACE "[\\\\/:*?\"<>|]" "_" SAFE_REF "${HALIDE_LLVM_REF}")
set(LLVM_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/src_cache/${SAFE_REF}")

# Validate that the source was downloaded by the version provider
if (NOT EXISTS "${LLVM_SOURCE_DIR}/llvm/CMakeLists.txt")
    message(FATAL_ERROR
        "LLVM source for ref '${HALIDE_LLVM_REF}' not found at:\n"
        "  ${LLVM_SOURCE_DIR}\n"
        "The Python version provider should have downloaded it.\n"
        "Try running: HALIDE_LLVM_REF=${HALIDE_LLVM_REF} pip wheel ."
    )
endif ()

message(STATUS "halide-llvm: Configuring LLVM from ${LLVM_SOURCE_DIR}")

# =============================================================================
# 3. Apply LLVM Configuration from Toolchain
# =============================================================================
# The toolchain file (via CMAKE_TOOLCHAIN_FILE) should have already included
# toolchains/initial-cache.cmake which sets all the LLVM_* variables.
#
# If no toolchain was specified, include the initial cache directly.
if (NOT CMAKE_TOOLCHAIN_FILE)
    message(STATUS "halide-llvm: No toolchain file specified, using initial-cache.cmake")
    include("${CMAKE_CURRENT_SOURCE_DIR}/toolchains/initial-cache.cmake")
endif ()

# =============================================================================
# 4. Build zlib
# =============================================================================
# LLVM is built with zlib support (LLVM_ENABLE_ZLIB=FORCE_ON), against a static
# zlib that the wheel ships next to the LLVM libraries. LLVM looks for it with
# find_package(ZLIB) while it is configured, so it is built right now, into a
# staging prefix that holds only the static library and the public headers.
set(HALIDE_LLVM_ZLIB_VERSION "1.3.1")
set(HALIDE_LLVM_ZLIB_SHA256 "9a93b2b7dfdac77ceba5a558a580e74667dd6fede4585b91eefb60f03b72df23")

set(zlib_dir "${CMAKE_CURRENT_BINARY_DIR}/zlib")
set(ZLIB_STAGE_DIR "${zlib_dir}/stage")
if (MSVC)
    set(zlib_built_name "zlibstatic.lib")
    set(zlib_staged_name "zlib.lib")
else ()
    set(zlib_built_name "libz.a")
    set(zlib_staged_name "libz.a")
endif ()

if (NOT EXISTS "${ZLIB_STAGE_DIR}/lib/${zlib_staged_name}")
    message(STATUS "halide-llvm: Building zlib ${HALIDE_LLVM_ZLIB_VERSION}")

    set(zlib_archive "${zlib_dir}/zlib-${HALIDE_LLVM_ZLIB_VERSION}.tar.gz")
    file(DOWNLOAD
        "https://github.com/madler/zlib/releases/download/v${HALIDE_LLVM_ZLIB_VERSION}/zlib-${HALIDE_LLVM_ZLIB_VERSION}.tar.gz"
        "${zlib_archive}"
        EXPECTED_HASH "SHA256=${HALIDE_LLVM_ZLIB_SHA256}"
        STATUS zlib_download_status
    )
    list(GET zlib_download_status 0 zlib_download_code)
    if (NOT zlib_download_code EQUAL 0)
        message(FATAL_ERROR "Failed to download zlib: ${zlib_download_status}")
    endif ()
    file(ARCHIVE_EXTRACT INPUT "${zlib_archive}" DESTINATION "${zlib_dir}")

    set(zlib_src "${zlib_dir}/zlib-${HALIDE_LLVM_ZLIB_VERSION}")
    set(zlib_build "${zlib_dir}/build")
    set(zlib_args
        -G "${CMAKE_GENERATOR}"
        -DCMAKE_BUILD_TYPE=Release
        "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
        -DCMAKE_POSITION_INDEPENDENT_CODE=ON
        # zlib 1.3.1 asks for CMake 2.4.4, which CMake 4 no longer accepts.
        -DCMAKE_POLICY_VERSION_MINIMUM=3.5
        -DZLIB_BUILD_EXAMPLES=OFF
    )
    foreach (var IN ITEMS CMAKE_MAKE_PROGRAM CMAKE_C_FLAGS CMAKE_OSX_ARCHITECTURES CMAKE_OSX_DEPLOYMENT_TARGET)
        if (${var})
            list(APPEND zlib_args "-D${var}=${${var}}")
        endif ()
    endforeach ()

    execute_process(
        COMMAND "${CMAKE_COMMAND}" -S "${zlib_src}" -B "${zlib_build}" ${zlib_args}
        COMMAND_ERROR_IS_FATAL ANY
    )
    execute_process(
        COMMAND "${CMAKE_COMMAND}" --build "${zlib_build}" --target zlibstatic --config Release
        COMMAND_ERROR_IS_FATAL ANY
    )

    # zlib's own install step would also install its shared library, so copy
    # what we need by hand. zconf.h is generated in the build directory.
    file(MAKE_DIRECTORY "${ZLIB_STAGE_DIR}/include" "${ZLIB_STAGE_DIR}/lib")
    file(COPY_FILE "${zlib_src}/zlib.h" "${ZLIB_STAGE_DIR}/include/zlib.h")
    file(COPY_FILE "${zlib_build}/zconf.h" "${ZLIB_STAGE_DIR}/include/zconf.h")
    file(COPY_FILE "${zlib_src}/LICENSE" "${ZLIB_STAGE_DIR}/LICENSE")
    file(COPY_FILE "${zlib_build}/${zlib_built_name}" "${ZLIB_STAGE_DIR}/lib/${zlib_staged_name}")
endif ()

set(ZLIB_ROOT "${ZLIB_STAGE_DIR}")
set(ZLIB_USE_STATIC_LIBS ON)

install(FILES "${ZLIB_STAGE_DIR}/include/zlib.h" "${ZLIB_STAGE_DIR}/include/zconf.h" DESTINATION include)
install(FILES "${ZLIB_STAGE_DIR}/lib/${zlib_staged_name}" DESTINATION lib)
install(FILES "${ZLIB_STAGE_DIR}/LICENSE" DESTINATION share/doc/zlib)

# =============================================================================
# 5. Build LLVM
# =============================================================================
add_subdirectory("${LLVM_SOURCE_DIR}/llvm" llvm-build)

# =============================================================================
# 6. Post-install fixup: point LLVMConfig.cmake at the shipped zlib
# =============================================================================
# LLVMConfig.cmake calls find_package(ZLIB) for the ZLIB::ZLIB target that
# LLVMSupport links to, after setting ZLIB_ROOT to the value it had while LLVM
# was configured: the staging prefix above. Make it the installed prefix.
install(CODE "set(ZLIB_STAGE_DIR \"${ZLIB_STAGE_DIR}\")")
install(CODE [[
    set(config "${CMAKE_INSTALL_PREFIX}/lib/cmake/llvm/LLVMConfig.cmake")
    file(READ "${config}" contents)
    set(staged_root "set(ZLIB_ROOT ${ZLIB_STAGE_DIR})")
    string(FIND "${contents}" "${staged_root}" at)
    if (at EQUAL -1)
        message(FATAL_ERROR "Did not find '${staged_root}' in ${config}")
    endif ()
    string(REPLACE "${staged_root}" "set(ZLIB_ROOT \"\${LLVM_INSTALL_PREFIX}\")" contents "${contents}")
    file(WRITE "${config}" "${contents}")
]])

# =============================================================================
# 7. Post-install cleanup: convert clang symlinks to wrappers
# =============================================================================
# LLVM creates symlinks like clang++ -> clang, but wheels are ZIP files that
# don't support symlinks. Each "symlink" becomes a full ~128MB copy.
# We replace them with small bash wrappers to save space while keeping functionality.
install(CODE "set(CONVERT_SYMLINK_SCRIPT \"${CMAKE_CURRENT_SOURCE_DIR}/convert-symlink.sh\")")
install(CODE [[
    file(GLOB binaries "${CMAKE_INSTALL_PREFIX}/bin/*")
    foreach (binary IN LISTS binaries)
        if (IS_SYMLINK "${binary}")
            message(STATUS "Converting symlink: ${binary}")
            execute_process(
                COMMAND bash "${CONVERT_SYMLINK_SCRIPT}" "${binary}"
                RESULT_VARIABLE res
                ERROR_VARIABLE err
            )
            if (NOT res EQUAL 0)
                message(FATAL_ERROR "Failed to convert symlink: ${err}")
            endif ()
        endif ()
    endforeach ()
]])