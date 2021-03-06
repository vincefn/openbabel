# define TESTDATADIR for tests that need input files
add_definitions(-DTESTDATADIR="\\"${CMAKE_SOURCE_DIR}/test/files/\\"")

# define FORMATDIR for location of format plugin binaries
add_definitions(-DFORMATDIR="\\"${openbabel_BINARY_DIR}/lib${LIB_SUFFIX}/\\"")

###########################################################
#  new tests using obtest.h
############################################################

set(tests
    automorphism
    builder
    canonconsistent
    canonstable
    cistrans
    graphsym
    implicitH
    lssr
    isomorphism
    rotor
    shuffle
    smiles
    smiley
    spectrophore
    squareplanar
    stereo
    stereoperception
    tautomer
    tetrahedral
    tetranonplanar
    tetraplanar
    uniqueid
  )

if(EIGEN2_FOUND OR EIGEN3_FOUND)
  set(tests
      align
      ${tests}
     )
endif()

set(TEST_PATH ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})

if(NOT BUILD_SHARED)
  set(libs openbabel)
  if(WITH_STATIC_LIBXML)
    set(libs ${libs} ${LIBXML2_LIBRARIES})
  endif()
  if(WITH_STATIC_INCHI)
    set(libs ${libs} ${INCHI_LIBRARY})
  endif()
  if(NOT MSVC)
    if(BUILD_MIXED)
      if(CMAKE_COMPILER_IS_GNUCXX)
        if(CMAKE_SYSTEM_NAME MATCHES Linux)

          #
          # Relocatable binaries on linux using -static-libgcc. (GCC only)
          #
          set(CMAKE_CXX_FLAGS "-static-libgcc ${CMAKE_CXX_FLAGS}")
          # make sure the linker will find the static libstdc++
          file(REMOVE "${CMAKE_BINARY_DIR}/libstdc++.a")
          execute_process(COMMAND "g++" "-print-file-name=libstdc++.a"
                          OUTPUT_VARIABLE libstdcpp
                          OUTPUT_STRIP_TRAILING_WHITESPACE)
          execute_process(COMMAND "ln" "-s" "${libstdcpp}"
                          WORKING_DIRECTORY ${CMAKE_BINARY_DIR})

          link_directories(${CMAKE_BINARY_DIR})

        endif(CMAKE_SYSTEM_NAME MATCHES Linux)
      endif(CMAKE_COMPILER_IS_GNUCXX)
    else(BUILD_MIXED)
      if(NOT APPLE)
        set(CMAKE_CXX_FLAGS "-static ${CMAKE_CXX_FLAGS}")
      endif()
    endif(BUILD_MIXED)
  endif()
else()
  set(libs openbabel)
endif()

foreach(test ${tests})
  add_executable(test_${test} ${test}test.cpp)
  target_link_libraries(test_${test} ${libs})
  if(NOT BUILD_SHARED AND NOT BUILD_MIXED)
    set_target_properties(test_${test} PROPERTIES LINK_SEARCH_END_STATIC TRUE)
  endif()
  add_test(${test}_Test ${TEST_PATH}/test_${test})
  set_tests_properties(${test}_Test PROPERTIES
    FAIL_REGULAR_EXPRESSION "ERROR;FAIL;Test failed"
    ENVIRONMENT "BABEL_DATADIR=${CMAKE_SOURCE_DIR}/data;LD_LIBRARY_PATH=${CMAKE_BINARY_DIR}/lib${LIB_SUFFIX}")
endforeach(test ${tests})

############################################################
#  old-style tests using "not ok"
############################################################

set(old_tests
    aromatest
    atom
    bond
    cansmi
    charge-mmff94
    charge-gasteiger
    conversion
    datatest
    ffgaff
    ffghemical
    ffmmff94
    ffuff
    formalcharge
    format
    formula
    internalcoord
    invalidsmarts
    invalidsmiles
    iterators
    logp_psa
    math
    mol
    phmodel
    residue
    ringtest
    smartstest
    smartsparse
    smilesmatch
    unitcell
   )
if(BUILD_SHARED)
  if(LIBXML2_FOUND)
    set(oldtests ${oldtests} cmlreadfile)
  endif(LIBXML2_FOUND)
else()
  if(WITH_STATIC_LIBXML)
    set(oldtests ${oldtests} cmlreadfile)
  endif()
  set(WITH_INCHI ${WITH_STATIC_INCHI})
endif()

if(WITH_INCHI)
  add_executable(test_inchiwrite inchiwrite.cpp ../src/formats/getinchi.cpp)
  target_link_libraries(test_inchiwrite ${libs})
  if(NOT BUILD_SHARED AND NOT BUILD_MIXED)
    set_target_properties(test_inchiwrite PROPERTIES LINK_SEARCH_END_STATIC TRUE)
  endif()
  # files in test/inchi -- both .sdf and .txt
  set(inchitests Samples.sdf SamplesTechMan.sdf Steffen_PubChem.smi)
  set(inchidata ${CMAKE_SOURCE_DIR}/test/inchi)
  foreach(test ${inchitests})
    add_test(inchi${test}_Test
             ${TEST_PATH}/test_inchiwrite ${inchidata}/${test} ${inchidata}/${test}.txt)
    set_tests_properties(inchi${test}_Test PROPERTIES
                         FAIL_REGULAR_EXPRESSION "Not ok"
                         ENVIRONMENT "BABEL_DATADIR=${CMAKE_SOURCE_DIR}/data;LD_LIBRARY_PATH=${CMAKE_BINARY_DIR}/lib${LIB_SUFFIX}"
    )
  endforeach(test ${inchitests})
endif(WITH_INCHI)

foreach(test ${old_tests})
  add_executable(test_${test} ${test}.cpp)
  target_link_libraries(test_${test} ${libs})
  if(NOT BUILD_SHARED AND NOT BUILD_MIXED)
    set_target_properties(test_${test} PROPERTIES LINK_SEARCH_END_STATIC TRUE)
  endif()
  add_test(${test}_Test ${TEST_PATH}/test_${test})
  set_tests_properties(${test}_Test PROPERTIES
    FAIL_REGULAR_EXPRESSION "not ok"
    ENVIRONMENT "BABEL_DATADIR=${CMAKE_SOURCE_DIR}/data;LD_LIBRARY_PATH=${CMAKE_BINARY_DIR}/lib${LIB_SUFFIX}"
 )
endforeach(test ${old_tests})

############################################################
#  roundtrip
############################################################

add_executable(roundtrip roundtrip.cpp)
target_link_libraries(roundtrip ${libs})
if(NOT BUILD_SHARED AND NOT BUILD_MIXED)
  set_target_properties(roundtrip PROPERTIES LINK_SEARCH_END_STATIC TRUE)
endif()
install(TARGETS roundtrip
                RUNTIME DESTINATION bin
                LIBRARY DESTINATION lib${LIB_SUFFIX}
                ARCHIVE DESTINATION lib${LIB_SUFFIX}
)

###########################
# Tests wrapped in Python #
###########################

# The following tests should work okay in MinGW, it's just that the calling
# script needs some work. On Cygwin, there's some weird DLL problems when
# calling from Python.
if(NOT MINGW AND NOT CYGWIN)
  include(UsePythonTest)
  if(PYTHON_EXECUTABLE)
    set(pytests
        babel sym smartssym fastsearch unique kekule)
    foreach(pytest ${pytests})
    SET_SOURCE_FILES_PROPERTIES(test${pytest}.py PROPERTIES
      PYTHONPATH "${CMAKE_SOURCE_DIR}/scripts/python:${CMAKE_BINARY_DIR}/lib${LIB_SUFFIX}"
      BABEL_LIBDIR "${CMAKE_BINARY_DIR}/lib${LIB_SYFFIX}"
      BABEL_DATADIR "${CMAKE_SOURCE_DIR}/data"
      LD_LIBRARY_PATH "${CMAKE_BINARY_DIR}/lib${LIB_SUFFIX}"
    )
      ADD_PYTHON_TEST(pytest_${pytest} test${pytest}.py)
      set_tests_properties(pytest_${pytest} PROPERTIES
           FAIL_REGULAR_EXPRESSION "ERROR;FAIL;Test failed"
      )
    endforeach(pytest ${pytests})
  endif(PYTHON_EXECUTABLE)
endif(NOT MINGW AND NOT CYGWIN)

###############################
# Tests using Python bindings #
###############################

if (PYTHON_BINDINGS)
  set(pybindtests
      bindings _pybel example)
  foreach(pybindtest ${pybindtests})
    SET_SOURCE_FILES_PROPERTIES(test${pybindtest}.py PROPERTIES
        PYTHONPATH "${CMAKE_SOURCE_DIR}/scripts/python:${CMAKE_BINARY_DIR}/lib${LIB_SUFFIX}"
        BABEL_LIBDIR "${CMAKE_BINARY_DIR}/lib${LIB_SYFFIX}"
        BABEL_DATADIR "${CMAKE_SOURCE_DIR}/data"
        LD_LIBRARY_PATH "${CMAKE_BINARY_DIR}/lib${LIB_SUFFIX}"
    )
    if(MSVC)
      SET_SOURCE_FILES_PROPERTIES(test${pybindtest}.py PROPERTIES
        PYTHONPATH "\"${CMAKE_SOURCE_DIR}/scripts/python;${CMAKE_BINARY_DIR}/bin/Release\""
      )
    endif()
    ADD_PYTHON_TEST(pybindtest_${pybindtest} test${pybindtest}.py)
    set_tests_properties(pybindtest_${pybindtest} PROPERTIES
         FAIL_REGULAR_EXPRESSION "ERROR;FAIL;Test failed"
    )
  endforeach(pybindtest ${pybindtests})
endif (PYTHON_BINDINGS)

############################################################
# Test 2D depiction using ImageMagick utilities
############################################################

# Example: add_depict_test(${CMAKE_SOURCE_DIR}/test/files/aromatics.smi aromatics ${CMAKE_SOURCE_DIR}/test/files/alias.png)
function(add_depict_test testname obfile image_file ref_image)
  set(depict_dir ${CMAKE_BINARY_DIR}/depict_test)
  add_test(NAME test_${testname} COMMAND ${CMAKE_SOURCE_DIR}/test/test_depiction.sh
      ${CMAKE_BINARY_DIR}/bin/babel ${ImageMagick_convert_EXECUTABLE}
      ${ImageMagick_compare_EXECUTABLE} ${obfile} ${depict_dir}
      ${image_file} ${ref_image})
  set_tests_properties(test_${testname} PROPERTIES
      PASS_REGULAR_EXPRESSION "0 @ 0,0")
endfunction()

option(TEST_DEPICTION "Test depiction using ImageMagick utilities" OFF)
if(TEST_DEPICTION AND UNIX)
  find_package(ImageMagick COMPONENTS convert compare)
  if(ImageMagick_convert_FOUND AND ImageMagick_compare_FOUND)
    set(depict_dir ${CMAKE_BINARY_DIR}/depict_test)
    file(MAKE_DIRECTORY ${depict_dir})
    add_depict_test(depict_aromatics ${CMAKE_SOURCE_DIR}/test/files/aromatics.smi
        aromatics ${CMAKE_SOURCE_DIR}/test/files/aromatics.png)
    add_depict_test(depict_alias ${CMAKE_SOURCE_DIR}/test/files/alias.mol
        alias ${CMAKE_SOURCE_DIR}/test/files/alias.png)
  endif()
endif()
