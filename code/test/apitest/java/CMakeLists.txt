# Copyright (c) 2005-2009 Jaroslav Gresula
#
# Distributed under the MIT license (See accompanying file
# LICENSE.txt or copy at http://jagpdf.org/LICENSE.txt)
#


if(JAG_STANDALONE_APITESTS)
  if(NOT JAG_APITEST_JAVA)
    return()
  else()
    find_package(Java)
    if(NOT JAVA_COMPILE)
      message(FATAL_ERROR "Cannot run tests: Java environment not found.")
    endif()
    message(STATUS "Java compiler: ${JAVA_COMPILE}")
    message(STATUS "Java runtime:  ${JAVA_RUNTIME}")
  endif()
endif()

#
# common variables
#
set(TEST_PDF_OUTPUT_DIR ${CMAKE_BINARY_DIR}/pdf-output/java)
file(MAKE_DIRECTORY ${TEST_PDF_OUTPUT_DIR})
set_property(DIRECTORY APPEND PROPERTY TEST_INCLUDE_FILE ${JAG_TEST_ENV})
set(JAGPDF_JAVA_JAR_DIST "${JAG_JAVA_LIB_LOCATION}/${JAGPDF_JAVA_JAR}")
set(ARG_CLASSPATH "${JAGPDF_JAVA_JAR_DIST}${PATH_SEP}${CMAKE_CURRENT_BINARY_DIR}")

#
# list of tests
#
set(JAVA_FILE_STEMS
  # tests
  basic
  helloworld
  externalstream
  # new to be verified
  basic_text
  basic_image
  encrypted
  docexamples
)

#
# creates a command compiling .java -> .class, establishes dependency on testlib
#
macro(jag_java_compile_test STEM CLASS_LIST)
  SET(JAVA_CLASS "${CMAKE_CURRENT_BINARY_DIR}/${STEM}.class")
  SET(JAVA_SRC "${CMAKE_CURRENT_SOURCE_DIR}/${STEM}.java")
  list(APPEND ${CLASS_LIST} "${JAVA_CLASS}")
  add_custom_command(
    OUTPUT ${JAVA_CLASS}
    COMMAND "${JAVA_COMPILE}" -source 1.4 -d ${CMAKE_CURRENT_BINARY_DIR} -classpath "${ARG_CLASSPATH}" ${JAVA_SRC}
    DEPENDS ${JAVA_SRC};${CMAKE_CURRENT_BINARY_DIR}/testlib/testlib.class
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    )
endmacro()

#
# creates a command compiling a package
#
macro(jag_java_compile_package STEM)
  SET(JAVA_CLASS "${CMAKE_CURRENT_BINARY_DIR}/${STEM}/${STEM}.class")
  SET(JAVA_SRC "${CMAKE_CURRENT_SOURCE_DIR}/${STEM}.java")
  add_custom_command(
    OUTPUT ${JAVA_CLASS}
    COMMAND "${JAVA_COMPILE}" -source 1.4 -d ${CMAKE_CURRENT_BINARY_DIR} -classpath "${ARG_CLASSPATH}" ${JAVA_SRC}
    DEPENDS ${JAVA_SRC}
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    )
endmacro()



# common testlib
jag_java_compile_package(testlib)

# tests
foreach(STEM ${JAVA_FILE_STEMS})
  jag_java_compile_test(${STEM} JAVA_TEST_CLASSES)
  add_test(api_java_${STEM} ${JAVA_RUNTIME} -enableassertions -classpath "${ARG_CLASSPATH}" ${STEM} ${TEST_PDF_OUTPUT_DIR})
endforeach(STEM)

# the infinite test
jag_java_compile_test(long_test JAVA_LONG_TEST_CLASS)
add_test(api_infinite_java ${JAVA_RUNTIME} -enableassertions -classpath "${ARG_CLASSPATH}" long_test ${TEST_PDF_OUTPUT_DIR})
set_tests_properties(api_infinite_java PROPERTIES TIMEOUT 1000000)


#
# the following macro generates targets for individual test types
#
macro(apitests_java_generic NAME CLASSES TEST_REX MEMCHECK)
  add_custom_target(${NAME}
    DEPENDS ${CLASSES}
    COMMAND ${CMAKE_COMMAND} -E remove ${TEST_PDF_OUTPUT_DIR}/*.pdf
    COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_SOURCE_DIR}/.outfiles ${TEST_PDF_OUTPUT_DIR}/.outfiles
    COMMAND ${JAG_MYCTEST_SH} ${MEMCHECK} ${CMAKE_CTEST_COMMAND} -R ${TEST_REX} ${ARGN}
    COMMAND ${CMAKE_COMMAND} -E echo "-- Verifying PDF files generated by target apitests-java"
    COMMAND ${JAG_COMPARE_PDF} --verify-list --out-dir=${TEST_PDF_OUTPUT_DIR} --good-dir=${PDF_REFERENCE_DIR}
    )
endmacro(apitests_java_generic)


#
# instantiate test targets
#
apitests_java_generic(apitests-java "${JAVA_TEST_CLASSES}" api_java_ "")
apitests_java_generic(apitests-java-memcheck "${JAVA_TEST_CLASSES}" api_java_ "")
apitests_java_generic(apitests-infinite-java "${JAVA_LONG_TEST_CLASS}" api_infinite_java "")

#java and valgrind:http://valgrind.org/docs/manual/faq.html#faq.java
#apitests_java_generic(apitests-java-memcheck "${JAVA_TEST_CLASSES}" api_java_ "${JAG_HAVE_MEMCHECK}")

add_dependencies(apitests apitests-java)
add_dependencies(apitests-memcheck apitests-java-memcheck)


# there is no way how to get an environment variable in Java 1.4, so we write
# path to the resource dir to file JAG_TEST_RESOURCES_DIR.env
# http://java.sun.com/j2se/1.4.2/docs/api/java/lang/System.html#getenv(java.lang.String)
file(WRITE
  ${CMAKE_CURRENT_BINARY_DIR}/JAG_TEST_RESOURCES_DIR.env
  "${CMAKE_CURRENT_SOURCE_DIR}/../resources")
