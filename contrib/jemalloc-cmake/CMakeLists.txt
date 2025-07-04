if (SANITIZE OR NOT (
    ((OS_LINUX OR OS_FREEBSD) AND (ARCH_AMD64 OR ARCH_AARCH64 OR ARCH_PPC64LE OR ARCH_RISCV64 OR ARCH_S390X)) OR
    (OS_DARWIN AND (CMAKE_BUILD_TYPE_UC STREQUAL "RELWITHDEBINFO" OR CMAKE_BUILD_TYPE_UC STREQUAL "DEBUG"))
))
    if (ENABLE_JEMALLOC)
        message (${RECONFIGURE_MESSAGE_LEVEL}
                 "jemalloc is disabled implicitly: it doesn't work with sanitizers and can only be used with x86_64, aarch64, or ppc64le Linux or FreeBSD builds and RelWithDebInfo macOS builds. Use -DENABLE_JEMALLOC=0")
    endif ()
    set (ENABLE_JEMALLOC OFF)
else ()
    option (ENABLE_JEMALLOC "Enable jemalloc allocator" ${ENABLE_LIBRARIES})
endif ()

if (NOT ENABLE_JEMALLOC)
    message (STATUS "Not using jemalloc")
    return()
endif ()

if (NOT OS_LINUX)
    message (WARNING "jemalloc support on non-Linux is EXPERIMENTAL")
endif()

if (OS_LINUX)
    # ThreadPool select job randomly, and there can be some threads that have been
    # performed some memory-heavy tasks before and will be inactive for some time,
    # but until it becomes active again, the memory will not be freed since, by
    # default, each thread has its arena, but there should be no more than
    # 4*CPU arenas (see opt.nareans description).
    #
    # By enabling percpu_arena number of arenas is limited to the number of CPUs, and hence
    # this problem should go away.
    #
    # muzzy_decay_ms -- use MADV_FREE when available on newer Linuxes, to
    # avoid spurious latencies and additional work associated with
    # MADV_DONTNEED. See
    # https://github.com/ClickHouse/ClickHouse/issues/11121 for motivation.

    # chdb: set background_thread true will cause Python 3.8 assert error on Linux aarch64:
    #   <jemalloc>: contrib/jemalloc/src/jemalloc.c:4201: Failed assertion: "malloc_initialized()"
    # so, disable it for now.
    set (JEMALLOC_CONFIG_MALLOC_CONF "percpu_arena:percpu,oversize_threshold:0,muzzy_decay_ms:0,dirty_decay_ms:5000,prof:true,prof_active:false,background_thread:false")
else()
    set (JEMALLOC_CONFIG_MALLOC_CONF "oversize_threshold:0,muzzy_decay_ms:0,dirty_decay_ms:5000")
endif()
# CACHE variable is empty to allow changing defaults without the necessity
# to purge cache
set (JEMALLOC_CONFIG_MALLOC_CONF_OVERRIDE "" CACHE STRING "Change default configuration string of JEMalloc" )
if (JEMALLOC_CONFIG_MALLOC_CONF_OVERRIDE)
    set (JEMALLOC_CONFIG_MALLOC_CONF "${JEMALLOC_CONFIG_MALLOC_CONF_OVERRIDE}")
endif()
message (STATUS "jemalloc malloc_conf: ${JEMALLOC_CONFIG_MALLOC_CONF}")

set (LIBRARY_DIR "${ClickHouse_SOURCE_DIR}/contrib/jemalloc")

set (SRCS
    "${LIBRARY_DIR}/src/arena.c"
    "${LIBRARY_DIR}/src/background_thread.c"
    "${LIBRARY_DIR}/src/base.c"
    "${LIBRARY_DIR}/src/bin.c"
    "${LIBRARY_DIR}/src/bin_info.c"
    "${LIBRARY_DIR}/src/bitmap.c"
    "${LIBRARY_DIR}/src/buf_writer.c"
    "${LIBRARY_DIR}/src/cache_bin.c"
    "${LIBRARY_DIR}/src/ckh.c"
    "${LIBRARY_DIR}/src/counter.c"
    "${LIBRARY_DIR}/src/ctl.c"
    "${LIBRARY_DIR}/src/decay.c"
    "${LIBRARY_DIR}/src/div.c"
    "${LIBRARY_DIR}/src/ecache.c"
    "${LIBRARY_DIR}/src/edata.c"
    "${LIBRARY_DIR}/src/edata_cache.c"
    "${LIBRARY_DIR}/src/ehooks.c"
    "${LIBRARY_DIR}/src/emap.c"
    "${LIBRARY_DIR}/src/eset.c"
    "${LIBRARY_DIR}/src/exp_grow.c"
    "${LIBRARY_DIR}/src/extent.c"
    "${LIBRARY_DIR}/src/extent_dss.c"
    "${LIBRARY_DIR}/src/extent_mmap.c"
    "${LIBRARY_DIR}/src/fxp.c"
    "${LIBRARY_DIR}/src/hook.c"
    "${LIBRARY_DIR}/src/hpa.c"
    "${LIBRARY_DIR}/src/hpa_hooks.c"
    "${LIBRARY_DIR}/src/hpdata.c"
    "${LIBRARY_DIR}/src/inspect.c"
    "${LIBRARY_DIR}/src/jemalloc.c"
    "${LIBRARY_DIR}/src/large.c"
    "${LIBRARY_DIR}/src/log.c"
    "${LIBRARY_DIR}/src/malloc_io.c"
    "${LIBRARY_DIR}/src/mutex.c"
    "${LIBRARY_DIR}/src/nstime.c"
    "${LIBRARY_DIR}/src/pa.c"
    "${LIBRARY_DIR}/src/pac.c"
    "${LIBRARY_DIR}/src/pa_extra.c"
    "${LIBRARY_DIR}/src/pages.c"
    "${LIBRARY_DIR}/src/pai.c"
    "${LIBRARY_DIR}/src/peak_event.c"
    "${LIBRARY_DIR}/src/prof.c"
    "${LIBRARY_DIR}/src/prof_data.c"
    "${LIBRARY_DIR}/src/prof_log.c"
    "${LIBRARY_DIR}/src/prof_recent.c"
    "${LIBRARY_DIR}/src/prof_stats.c"
    "${LIBRARY_DIR}/src/prof_sys.c"
    "${LIBRARY_DIR}/src/psset.c"
    "${LIBRARY_DIR}/src/rtree.c"
    "${LIBRARY_DIR}/src/safety_check.c"
    "${LIBRARY_DIR}/src/san_bump.c"
    "${LIBRARY_DIR}/src/san.c"
    "${LIBRARY_DIR}/src/sc.c"
    "${LIBRARY_DIR}/src/sec.c"
    "${LIBRARY_DIR}/src/stats.c"
    "${LIBRARY_DIR}/src/sz.c"
    "${LIBRARY_DIR}/src/tcache.c"
    "${LIBRARY_DIR}/src/test_hooks.c"
    "${LIBRARY_DIR}/src/thread_event.c"
    "${LIBRARY_DIR}/src/ticker.c"
    "${LIBRARY_DIR}/src/tsd.c"
    "${LIBRARY_DIR}/src/witness.c"
)
if (OS_DARWIN)
    list(APPEND SRCS "${LIBRARY_DIR}/src/zone.c")
endif ()

add_library(_jemalloc ${SRCS})

# First include jemalloc-cmake files, to override anything that jemalloc has.
# (for example if you were trying to build jemalloc inside contrib/jemalloc you
# will have some files that may be out of date)
target_include_directories(_jemalloc SYSTEM PUBLIC include)
target_include_directories(_jemalloc PRIVATE "${LIBRARY_DIR}/include")

set (JEMALLOC_INCLUDE_PREFIX)
# OS_
if (OS_LINUX)
    set (JEMALLOC_INCLUDE_PREFIX "include_linux")
elseif (OS_FREEBSD)
    set (JEMALLOC_INCLUDE_PREFIX "include_freebsd")
elseif (OS_DARWIN)
    set (JEMALLOC_INCLUDE_PREFIX "include_darwin")
else ()
    message (FATAL_ERROR "internal jemalloc: This OS is not supported")
endif ()
# ARCH_
if (ARCH_AMD64)
    if (USE_MUSL)
        set(JEMALLOC_INCLUDE_PREFIX "${JEMALLOC_INCLUDE_PREFIX}_x86_64_musl")
    else()
        set(JEMALLOC_INCLUDE_PREFIX "${JEMALLOC_INCLUDE_PREFIX}_x86_64")
    endif()
elseif (ARCH_AARCH64)
    set(JEMALLOC_INCLUDE_PREFIX "${JEMALLOC_INCLUDE_PREFIX}_aarch64")
elseif (ARCH_PPC64LE)
    set(JEMALLOC_INCLUDE_PREFIX "${JEMALLOC_INCLUDE_PREFIX}_ppc64le")
elseif (ARCH_RISCV64)
    set(JEMALLOC_INCLUDE_PREFIX "${JEMALLOC_INCLUDE_PREFIX}_riscv64")
elseif (ARCH_S390X)
    set(JEMALLOC_INCLUDE_PREFIX "${JEMALLOC_INCLUDE_PREFIX}_s390x")
else ()
    message (FATAL_ERROR "internal jemalloc: This arch is not supported")
endif ()

configure_file(${JEMALLOC_INCLUDE_PREFIX}/jemalloc/internal/jemalloc_internal_defs.h.in
    ${JEMALLOC_INCLUDE_PREFIX}/jemalloc/internal/jemalloc_internal_defs.h)
target_include_directories(_jemalloc SYSTEM PRIVATE
    "${CMAKE_CURRENT_BINARY_DIR}/${JEMALLOC_INCLUDE_PREFIX}/jemalloc/internal")

target_compile_definitions(_jemalloc PRIVATE -DJEMALLOC_NO_PRIVATE_NAMESPACE)

# Because our coverage callbacks call malloc, and recursive call of malloc could not work.
target_compile_options(_jemalloc PRIVATE ${WITHOUT_COVERAGE_FLAGS_LIST})

target_compile_definitions(_jemalloc PRIVATE -DJEMALLOC_PROF=1)

# jemalloc provides support two unwind flavors:
# - JEMALLOC_PROF_LIBUNWIND - unw_backtrace() - gnu libunwind (compatible with llvm libunwind)
# - JEMALLOC_PROF_LIBGCC - _Unwind_Backtrace() - the original HP libunwind and the one coming with gcc / g++ / libstdc++.
#
# But for JEMALLOC_PROF_LIBGCC it also calls _Unwind_Backtrace() during
# bootstraping of jemalloc, which may lead to deadlock, if the dlsym will do
# allocations somewhere (like glibc does prio 2.34, see [1]).
#
#  [1]: https://sourceware.org/git/?p=glibc.git;a=commit;h=fada9018199c21c469ff0e731ef75c6020074ac9
#
# And since ClickHouse unwind already supports unw_backtrace() we can safely
# switch to it to avoid this deadlock.
target_compile_definitions (_jemalloc PRIVATE -DJEMALLOC_PROF_LIBUNWIND=1)
target_link_libraries (_jemalloc PRIVATE unwind)

# for RTLD_NEXT
target_compile_options(_jemalloc PRIVATE -D_GNU_SOURCE)

add_library(ch_contrib::jemalloc ALIAS _jemalloc)