cmake_minimum_required(VERSION 3.16)
project(vcflib)

set(CMAKE_CXX_STANDARD 17)

include(ExternalProject)
include(FeatureSummary)

find_package(PkgConfig REQUIRED)
find_package(pybind11 CONFIG)

include(FindBZip2)
include(FindLibLZMA)
include(FindZLIB)
include(FindCURL) # for htslib

feature_summary(
  FATAL_ON_MISSING_REQUIRED_PACKAGES
  WHAT REQUIRED_PACKAGES_NOT_FOUND)

set(CMAKE_POSITION_INDEPENDENT_CODE ON) # for pybind11

# ---- Options

option(BUILD_DOC "Build documentation" ON)
option(OPENMP "Enable OpenMP" ON) # disabling does not work because of vcfwave
option(PROFILING "Enable profiling" OFF)
option(GPROF "Enable gprof profiling" OFF)
option(ASAN "Use address sanitiser" OFF)
option(ZIG "Disable using the zig compiler" ON)

include(CheckIPOSupported) # adds lto
check_ipo_supported(RESULT ipo_supported OUTPUT output)

# ---- Dependencies

if(OPENMP)
  include(FindOpenMP)
  SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
  SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif(OPENMP)

find_package(ZLIB)
set_package_properties(ZLIB PROPERTIES TYPE REQUIRED)
#find_package(LIBLZMA)
#set_package_properties(LIBLZMA PROPERTIES TYPE REQUIRED)
find_package(Threads)
set_package_properties(Threads PROPERTIES TYPE REQUIRED)

pkg_check_modules(HTSLIB htslib)   # Optionally builds from contrib/
pkg_check_modules(TABIXPP tabixpp) # Optionally builds from contrib/

# ---- Build switches
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ${ipo_supported})

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release CACHE STRING
          "Choose the type of build, options are: Release|Debug|RelWithDebInfo (for distros)." FORCE)
endif()

if (${CMAKE_BUILD_TYPE} MATCHES Release)
  set(EXTRA_FLAGS "-march=native -D_FILE_OFFSET_BITS=64")
  # set(CMAKE_CXX_FLAGS_RELEASE "-DNDEBUG") # reset CXX_FLAGS to replace -O3 with -Ofast
endif()

if ((${CMAKE_BUILD_TYPE} MATCHES Release) OR (${CMAKE_BUILD_TYPE} MATCHES RelWithDebInfo))
  set (CMAKE_C_FLAGS "${OpenMP_C_FLAGS} ${EXTRA_FLAGS}")
  set (CMAKE_CXX_FLAGS "${OpenMP_CXX_FLAGS} ${EXTRA_FLAGS}")
endif ()

if (${CMAKE_BUILD_TYPE} MATCHES "Debug")
  set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${EXTRA_FLAGS}")
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${EXTRA_FLAGS}")
  add_definitions(-Wfatal-errors)
endif ()

if (ASAN)
  set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address  -fno-omit-frame-pointer -fno-common")
  set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address  -fno-omit-frame-pointer -fno-common")
endif(ASAN)

if(OPENMP)
  # find_package(OpenMP REQUIRED)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif(OPENMP)

if(PROFILING)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g")
endif(PROFILING)

if(GPROF)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pg")
endif(GPROF)

# ---- Include files

include_directories(include)
include_directories(contrib/fastahack)
include_directories(contrib/intervaltree)
include_directories(contrib/smithwaterman)
include_directories(contrib/multichoose)
include_directories(contrib/filevercmp)

if(NOT HTSLIB_FOUND)
  set(HTSLIB_LOCAL contrib/tabixpp/htslib)
  set(TABIX_FOUND OFF) # also build tabixpp if htslib is missing
endif()

if (NOT TABIX_FOUND)
  set(TABIXPP_LOCAL contrib/tabixpp)
  include_directories(contrib/tabixpp)
  set(tabixpp_SOURCE
    contrib/tabixpp/tabix.cpp
  )
endif()

set(WFA_LOCAL contrib/WFA2-lib)

file(GLOB INCLUDES
  src/*.h*
  contrib/multichoose/*.h*
  contrib/intervaltree/*.h*
  contrib/smithwaterman/*.h*
  contrib/fastahack/*.h*
  contrib/filevercmp/*.h*
  )

set(vcflib_SOURCE
    src/vcf-c-api.cpp
    src/legacy.cpp
    src/Variant.cpp
    src/rnglib.cpp
    src/var.cpp
    src/pdflib.cpp
    src/cdflib.cpp
    src/split.cpp
    src/legacy.cpp
    src/rkmh.cpp
    src/murmur3.cpp
    src/LeftAlign.cpp
    src/cigar.cpp
    src/allele.cpp
    contrib/fastahack/Fasta.cpp
    contrib/smithwaterman/SmithWatermanGotoh.cpp
    contrib/smithwaterman/Repeats.cpp
    contrib/smithwaterman/IndelAllele.cpp
    contrib/smithwaterman/disorder.cpp
    contrib/smithwaterman/LeftAlign.cpp
    contrib/fsom/fsom.c
    contrib/filevercmp/filevercmp.c
)

if (TABIXPP_LOCAL) # add the tabixpp source file
    list(APPEND vcflib_SOURCE ${tabixpp_SOURCE})
endif()

add_library(vcflib STATIC
    ${vcflib_SOURCE}
    )

set(BINS
    vcfecho
    dumpContigsFromHeader
    bFst
    pVst
    hapLrt
    popStats
    wcFst
    iHS
    segmentFst
    segmentIhs
    genotypeSummary
    sequenceDiversity
    pFst
    smoother
    vcfld
    plotHaps
    abba-baba
    permuteGPAT++
    permuteSmooth
    normalize-iHS
    meltEHH
    vcfnullgenofields
    vcfaltcount
    vcfhetcount
    vcfhethomratio
    vcffilter
    vcf2tsv
    vcfgenotypes
    vcfannotategenotypes
    vcfcommonsamples
    vcfremovesamples
    vcfkeepsamples
    vcfsamplenames
    vcfgenotypecompare
    vcffixup
    vcfclassify
    vcfsamplediff
    vcfremoveaberrantgenotypes
    vcfrandom
    vcfparsealts
    vcfstats
    vcfflatten
    vcfprimers
    vcfnumalt
    vcfcleancomplex
    vcfintersect
    vcfannotate
    vcfallelicprimitives
    vcfwave
    vcfoverlay
    vcfaddinfo
    vcfkeepinfo
    vcfkeepgeno
    vcfafpath
    vcfcountalleles
    vcflength
    vcfdistance
    vcfrandomsample
    vcfentropy
    vcfglxgt
    vcfroc
    vcfcheck
    vcfstreamsort
    vcfuniq
    vcfuniqalleles
    vcfremap
    vcf2fasta
    vcfsitesummarize
    vcfbreakmulti
    vcfevenregions
    vcfcat
    vcfgenosummarize
    vcfgenosamplenames
    vcfgeno2haplo
    vcfleftalign
    vcfcombine
    vcfgeno2alleles
    vcfindex
    vcf2dag
    vcfsample2info
    vcfqual2info
    vcfinfo2qual
    vcfglbound
    vcfinfosummarize
)

if (ZIG)
  list(APPEND BINS vcfcreatemulti)
endif (ZIG)

set(SCRIPTS
    bed2region
    bgziptabix
    # plotBfst.R
    # plotHaplotypes.R
    # plotHapLrt.R
    # plotPfst.R
    # plot_roc.r
    # plotSmoothed.R
    # plotWCfst.R
    # plotXPEHH.R
    vcf2bed.py
    vcf2sqlite.py
    vcfbiallelic
    vcfclearid
    vcfclearinfo
    vcfcomplex
    vcffirstheader
    vcfgtcompare.sh
    vcfindelproximity
    vcfindels
    vcfjoincalls
    vcfmultiallelic
    vcfmultiway
    vcfmultiwayscripts
    vcfnobiallelicsnps
    vcfnoindels
    vcfnosnps
    vcfnulldotslashdot
    vcfplotaltdiscrepancy.r
    vcfplotaltdiscrepancy.sh
    vcfplotsitediscrepancy.r
    vcfplottstv.sh
    vcfprintaltdiscrepancy.r
    vcfprintaltdiscrepancy.sh
    vcfqualfilter
    vcfregionreduce
    vcfregionreduce_and_cut
    vcfregionreduce_pipe
    vcfregionreduce_uncompressed
    vcfremovenonATGC
    vcfsnps
    vcfsort
    vcf_strip_extra_headers
    vcfvarstats
    )

# ---- Get version

file (STRINGS "VERSION" BUILD_NUMBER)
add_definitions(-DVCFLIB_VERSION="${BUILD_NUMBER}")
add_definitions(-DVERSION="${BUILD_NUMBER}")


# ---- Build htslib
#
# Note by default we use the distributed htslib! These are
# the older instructions which allow for a local build from
# git submodules

if (HTSLIB_LOCAL)

  include_directories(${HTSLIB_LOCAL})

  set(flags "-O2 -g -fPIC")
  ExternalProject_Add(htslib-EXT
    SOURCE_DIR "${CMAKE_SOURCE_DIR}/${HTSLIB_LOCAL}"
    UPDATE_COMMAND autoreconf -i
    CONFIGURE_COMMAND ./configure --disable-s3
    INSTALL_COMMAND ""
    BUILD_IN_SOURCE ON
    BUILD_COMMAND $(MAKE) CFLAGS=${flags} lib-static
  )
  ExternalProject_Get_property(htslib-EXT SOURCE_DIR)
  ExternalProject_Get_property(htslib-EXT INSTALL_DIR)
  set(htslib_INCLUDE "${SOURCE_DIR}")
  set(htslib_static "${SOURCE_DIR}/libhts.a")
  set(htslib_lib "${htslib_static}")
  set(HTSLIB_LINK_LIBRARIES "${htslib_static}")

  add_custom_target(htslib
    DEPENDS htslib-EXT
    VERBATIM)

  # set_property(TARGET htslib PROPERTY IMPORTED_LOCATION ${hstlib_lib})
  # add_dependencies(htslib htslib-EXT)

  add_library(HTSLIB_LIBRARIES STATIC IMPORTED)
  set_target_properties(HTSLIB_LIBRARIES PROPERTIES IMPORTED_LOCATION ${htslib_lib})
  add_dependencies(HTSLIB_LIBRARIES htslib-EXT)

# If the user wants to configure our HTSlib to build with libddeflate, we need
# to make sure to link against libdeflate as a transitive dependency. To do
# that, pass -DHTSLIB_EXTRA_LIBS="-ldeflate" when configuring the project with
# cmake.
# TODO: Stop vendoring in htslib and just use find_package

  # set(HTSLIB_EXTRA_LIBS "-lcurl" CACHE STRING "Library flags needed to link with htslib's dependencies, for chosen configuration")
  # set_property(TARGET htslib PROPERTY INTERFACE_LINK_LIBRARIES ${HTSLIB_EXTRA_LIBS})

endif(HTSLIB_LOCAL)

# FIXME: hard-coded compile switches need to be fixed upstream
if (ASAN)
  set(wfa_MAKE_ARGS BUILD_TOOLS=0 BUILD_EXAMPLES=0 CC=gcc CC_FLAGS=-fPIC setup asan lib_wfa)
else()
  set(wfa_MAKE_ARGS BUILD_TOOLS=0 BUILD_EXAMPLES=0 CC=gcc CC_FLAGS=-fPIC setup lib_wfa)
endif()

# if ((${CMAKE_BUILD_TYPE} MATCHES Release) OR (${CMAKE_BUILD_TYPE} MATCHES RelWithDebInfo))
#   set(wfa_MAKE_ARGS ${wfa_MAKE_ARGS})
# endif()

ExternalProject_Add(wfa-EXT
    SOURCE_DIR "${CMAKE_SOURCE_DIR}/${WFA_LOCAL}"
    UPDATE_COMMAND ""
    CONFIGURE_COMMAND ""
    INSTALL_COMMAND ""
    BUILD_IN_SOURCE ON
    BUILD_ALWAYS ON
    BUILD_COMMAND $(MAKE) ${wfa_MAKE_ARGS}
)
ExternalProject_Get_property(wfa-EXT SOURCE_DIR)
set(WFA_INCLUDE_DIRS ${SOURCE_DIR})
set(WFA_LINK_LIBRARIES ${SOURCE_DIR}/lib/libwfa.a )
# set(WFA_LINK_LIBRARIES ${SOURCE_DIR}/lib/libwfa.a ${SOURCE_DIR}/lib/libwfacpp.a )
link_directories($SOURCE_DIR}/lib)
MESSAGE(STATUS ${WFA_LINK_LIBRARIES})

# add_library(WAFCPP_STATIC_LIB STATIC IMPORTED)
# set_target_properties(WAFCPP_STATIC_LIB PROPERTIES IMPORTED_LOCATION ${WFA_LINK_LIBRARIES})
# add_dependencies(WAFCPP_STATIC_LIB wfa-EXT)

add_custom_target(wfa
    DEPENDS wfa-EXT
    VERBATIM)

include_directories(${WFA_INCLUDE_DIRS})

# ZIG VCF imported library is part of VCFLIB source tree. We designate it an
# external project so we can run 'zig build'

if (ZIG)
ExternalProject_Add(ZIG-EXT
    SOURCE_DIR "${CMAKE_SOURCE_DIR}/src/zig"
    UPDATE_COMMAND ""
    CONFIGURE_COMMAND ""
    INSTALL_COMMAND ""
    BUILD_IN_SOURCE ON
    BUILD_ALWAYS ON
    BUILD_COMMAND ${CMAKE_SOURCE_DIR}/src/zig/compile.sh
)
ExternalProject_Get_property(ZIG-EXT SOURCE_DIR)
set(ZIG_INCLUDE_DIRS ${SOURCE_DIR})
set(ZIG_LINK_LIBRARIES ${SOURCE_DIR}/zig-out/lib/libzig.a)
# link_directories($SOURCE_DIR}/lib)
# MESSAGE(STATUS ${ZIG_LINK_LIBRARIES})

add_library(ZIGCPP_STATIC_LIB STATIC IMPORTED)
set_target_properties(ZIGCPP_STATIC_LIB PROPERTIES IMPORTED_LOCATION ${ZIG_LINK_LIBRARIES})
add_dependencies(ZIGCPP_STATIC_LIB ZIG-EXT vcf.zig)

# add_custom_target(ZIGLIB
#     DEPENDS ZIG-EXT
#     VERBATIM)

include_directories(${ZIG_INCLUDE_DIRS})

else (ZIG)
  set (ZIG_LINK_LIBRARIES )
endif (ZIG)

set(vcflib_DEPS
  CURL::libcurl
  wfa
  )

if (ZIG)
  list(APPEND vcflib_DEPS ZIG-EXT)
endif ()

if (HTSLIB_LOCAL)
  list(APPEND vcflib_DEPS htslib)
endif()

set(vcflib_LIBS
  ${HTSLIB_LIBRARIES}
  ${CURL_LIBRARIES}
  ${ZLIB_LIBRARIES}
  ${LIBLZMA_LIBRARIES}
  ${BZIP2_LIBRARIES}
  ${TABIXPP_LIBRARIES}
  ${CMAKE_THREAD_LIBS_INIT}
  ${WFA_LINK_LIBRARIES}
  ${HTSLIB_LINK_LIBRARIES}
)

if (ZIG)
  list(APPEND vcflib_LIBS ${ZIG_LINK_LIBRARIES})
endif()

# target_link_libraries(vcflib PUBLIC ${vcflib_LIBS})
add_dependencies(vcflib ${vcflib_DEPS})

# ---- Build all

if (NOT BUILD_ONLY_LIB)
  foreach(BIN ${BINS})
    add_executable(${BIN} src/${BIN}.cpp)
    add_dependencies(${BIN} vcflib)
    target_link_libraries(${BIN} PUBLIC ${vcflib_LIBS} vcflib)
  endforeach(BIN ${BINS})
  install(TARGETS ${BINS} RUNTIME DESTINATION bin)

  # ---- Copy scripts
  foreach(SCRIPT ${SCRIPTS})
    install(PROGRAMS ./scripts/${SCRIPT} DESTINATION bin RENAME ${SCRIPT})
  endforeach(SCRIPT ${SCRIPTS})

endif()

# ---- Python bindings - mostly for testing at this stage
pybind11_add_module(pyvcflib "${CMAKE_SOURCE_DIR}/src/pythonffi.cpp")
add_dependencies(pyvcflib ${vcflib_DEPS})
target_link_libraries(pyvcflib PUBLIC vcflib "${vcflib_LIBS}")
install(TARGETS pyvcflib LIBRARY DESTINATION lib)

# ---- Test

enable_testing()

function(add_pytest TEST_FILE)
  add_test(
      NAME ${TEST_FILE}
      COMMAND python3 tests/${TEST_FILE}.py
      WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/test
  )
  SET_TESTS_PROPERTIES(${TEST_FILE}
      PROPERTIES ENVIRONMENT "LSAN_OPTIONS=suppressions=../test/libasan-suppress.txt;PYTHONPATH=${PROJECT_SOURCE_DIR}/build;LD_LIBRARY_PATH=$ENV{LIBRARY_PATH}")
endfunction()

function(add_pydoctest TEST_FILE)
  add_test(
      NAME ${TEST_FILE}
      COMMAND python3 -m doctest -o NORMALIZE_WHITESPACE -o REPORT_UDIFF pytest/${TEST_FILE}.md
      WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/test
  )
  SET_TESTS_PROPERTIES(${TEST_FILE}
      # PROPERTIES ENVIRONMENT "ASAN_OPTIONS=detect_leaks=1:symbolize=1;LSAN_OPTIONS=verbosity=2:log_threads=1:suppressions=../test/libasan-suppress.txt")
      PROPERTIES ENVIRONMENT "LSAN_OPTIONS=suppressions=../test/libasan-suppress.txt;PYTHONPATH=${PROJECT_SOURCE_DIR}/build:${PROJECT_SOURCE_DIR}/test/pytest;LD_LIBRARY_PATH=$ENV{LIBRARY_PATH}")
endfunction()


function(add_doctest TEST_FILE)
  add_test(
      NAME ${TEST_FILE}
      COMMAND python3 -m doctest -o NORMALIZE_WHITESPACE -o REPORT_UDIFF ../${TEST_FILE}.md
      WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/test
  )
endfunction()

function(add_pydoctest_fullname TEST_FILE)
  add_test(
      NAME ${TEST_FILE}
      COMMAND python3 -m doctest -o NORMALIZE_WHITESPACE -o REPORT_UDIFF ${TEST_FILE}
      WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/test
  )
endfunction()

if (ZIG)
function(add_zigtest)
  add_test(
    NAME zigvcf
    COMMAND zig build test
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/src/zig
    )
endfunction()

  add_zigtest()
endif(ZIG)

add_pytest(realign)
add_pydoctest(pyvcflib)
add_pydoctest(vcflib-api)
add_pydoctest(vcf2tsv)
add_pydoctest(vcfallelicprimitives)
add_pydoctest(vcfwave)
if (ZIG)
  add_pydoctest(vcfcreatemulti)
endif (ZIG)

add_pydoctest(vcfnulldotslashdot)
add_doctest(doc/vcfintersect)
add_doctest(doc/vcffilter)
# add_pydoctest_fullname(../README.md)

# ---- Build docs
#
# Generates man pages for the python doctests. Don't need
# to run every time so it is a separate command. For pandoc logic see
# https://www.howtogeek.com/682871/how-to-create-a-man-page-on-linux/
#
# cmake --build . --target man ; cmake --install .

find_program(PANDOC pandoc)

set(pytest vcfnulldotslashdot vcfallelicprimitives vcfwave)

# vcfcreatemulti)

if (PANDOC)
    # note the option ALL which allows to build the docs together with the application
    add_custom_target( man
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        COMMAND ruby ./test/scripts/bin2md.rb --man test/scripts/bin2md-template.erb
        COMMAND ruby ./test/scripts/bin2md.rb --man --index
        # overwrite markdown from tests
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/vcfnulldotslashdot.md ./doc/vcfnulldotslashdot.md
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/vcfallelicprimitives.md ./doc/vcfallelicprimitives.md
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/vcfwave.md ./doc/vcfwave.md
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/vcfcreatemulti.md ./doc/vcfcreatemulti.md
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/vcflib-api.md ./doc/vcflib-api.md
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/pyvcflib.md ./doc/pyvcflib.md
        # generate man pages from markdown
        COMMAND ruby ./test/scripts/md2man
        # regenerate to allow for URLs in markdown docs
        COMMAND ruby ./test/scripts/bin2md.rb test/scripts/bin2md-template.erb
        COMMAND ruby ./test/scripts/bin2md.rb --index
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/vcfnulldotslashdot.md ./doc/vcfnulldotslashdot.md
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/vcfallelicprimitives.md ./doc/vcfallelicprimitives.md
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/vcfwave.md ./doc/vcfwave.md
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/vcfcreatemulti.md ./doc/vcfcreatemulti.md
        # Documented test cases for Python bindings
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/pyvcflib.md ./doc/pyvcflib.md
        COMMAND ${CMAKE_COMMAND} -E copy ./test/pytest/vcflib-api.md ./doc/vcflib-api.md
    )
else (PANDOC)
  message("Pandoc needs to be installed to generate the man pages")
endif (PANDOC)

# ---- Install

install(TARGETS vcflib ARCHIVE DESTINATION lib)

install(FILES ${INCLUDES} DESTINATION include)

install(DIRECTORY ${CMAKE_SOURCE_DIR}/man/ DESTINATION ${CMAKE_INSTALL_PREFIX}/man/man1)
