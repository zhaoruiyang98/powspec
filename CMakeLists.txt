cmake_minimum_required(VERSION 3.18)
project(POWSPEC
    LANGUAGES C
    VERSION 0.1.0
    DESCRIPTION "a C program for computing auto and cross power spectrum multipoles"
)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

set(SRCS
    src/cnvt_coord.c
    src/genr_mesh.c
    src/load_conf.c
    src/mp_template.c
    src/multipole.c
    src/powspec.c
    src/read_cata.c
    src/save_res.c
    lib/libast.c
    lib/libcfg.c
    io/ascii_fmtr.c
    io/read_ascii.c
    io/read_file.c
    io/write_ascii.c
    math/cspline.c
    math/legauss.c
)
set(INCL
    src/
    io/
    lib/
    math/
)
set(LIBS m)
set(DEFINES)

add_executable(POWSPEC ${SRCS})

# Setting for single precision density fields and FFT
option(SINGLE_PREC "whether to use single precision")

if(SINGLE_PREC)
    list(APPEND DEFINES SINGLE_PREC)
    message(STATUS "enable single precision")
else()
    message(STATUS "enable double precision")
endif()

# Settings for openmp
option(USEOPENMP "whether to use OpenMP" ON)

if(USEOPENMP)
    find_package(OpenMP REQUIRED)
    target_link_libraries(POWSPEC PRIVATE OpenMP::OpenMP_C)
    list(APPEND DEFINES OMP)
    message(STATUS "use openmp")
else()
    message(STATUS "disable openmp")
endif()

# Settings for FFTW
set(FFTW_LIB_DIR OFF CACHE STRING "The path to FFTW3 library directory")

if(SINGLE_PREC)
    set(fftw_name fftw3f)
    set(fftw_omp_name fftw3f_omp)
else()
    set(fftw_name fftw3)
    set(fftw_omp_name fftw3_omp)
endif()

# try to find the library
if(FFTW_LIB_DIR)
    find_library(fftw_lib_path ${fftw_name}
        PATHS ${FFTW_LIB_DIR}
        REQUIRED NO_DEFAULT_PATH
    )

    if(USEOPENMP)
        find_library(tmp ${fftw_omp_name}
            PATHS ${FFTW_LIB_DIR}
            REQUIRED NO_DEFAULT_PATH
        )
        list(APPEND fftw_lib_path ${tmp})
    endif()
else()
    find_library(fftw_lib_path ${fftw_name} REQUIRED)

    if(USEOPENMP)
        find_library(tmp ${fftw_omp_name} REQUIRED)
        list(APPEND fftw_lib_path ${tmp})
    endif()
endif()

list(APPEND LIBS ${fftw_lib_path})
message(STATUS "find fftw3 library: ${fftw_lib_path}")

set(FFTW_INCLUDE_DIR OFF CACHE STRING "The path to FFTW3 include directory")

if(FFTW_INCLUDE_DIR)
    find_path(fftw_include_path fftw3.h
        PATHS ${FFTW_INCLUDE_DIR}
        REQUIRED
        NO_DEFAULT_PATH
    )
else()
    find_path(fftw_include_path fftw3.h REQUIRED)
endif()

list(APPEND INCL ${fftw_include_path})
message(STATUS "find fftw3 include directory: ${fftw_include_path}")

target_compile_definitions(POWSPEC PRIVATE ${DEFINES})
target_include_directories(POWSPEC PRIVATE ${INCL})
target_link_libraries(POWSPEC PRIVATE ${LIBS})

# unfortunately powspec is not portable due to dependence of POSIX libraries
target_compile_options(POWSPEC PRIVATE
    $<$<OR:$<C_COMPILER_ID:GNU>,$<C_COMPILER_ID:Clang>>:-Wall>
    $<$<C_COMPILER_ID:MSVC>:SHELL:/W4>
)
