CMAKE_MINIMUM_REQUIRED(VERSION 2.8 FATAL_ERROR)
INCLUDE(InstallRequiredSystemLibraries)
PROJECT(VERSIONHEADER C)
FIND_PACKAGE(Git)
FILE(WRITE ${CMAKE_BINARY_DIR}/main.c
"\#include \"version.h\"
\#include <stdio.h>
int main(void)
{
  printf(\"%s\\n\", VERSION);
  return 0;
}")
FILE(WRITE ${CMAKE_BINARY_DIR}/version.h.in
"\#define VERSION \"@VERSION@\"\n"
)

IF(GIT_FOUND)
  SET(USE_GIT 1)
ELSE(GIT_FOUND)
  SET(USE_GIT 0)
ENDIF(GIT_FOUND)
FILE(WRITE ${CMAKE_BINARY_DIR}/Version.cmake
"IF(${USE_GIT} EQUAL 1)
  EXECUTE_PROCESS(
       COMMAND ${GIT_EXECUTABLE} describe --tags --dirty
       WORKING_DIRECTORY \${WORKING_DIRECTORY}
       OUTPUT_VARIABLE GIT_DESCRIBE_VERSION
       RESULT_VARIABLE GIT_DESCRIBE_RESULT
       ERROR_VARIABLE GIT_DESCRIBE_ERROR
       OUTPUT_STRIP_TRAILING_WHITESPACE
   )
ELSE(${USE_GIT} EQUAL 1)
  SET(GIT_DESCRIBE_RESULT -1)
ENDIF(${USE_GIT} EQUAL 1)

IF(GIT_DESCRIBE_RESULT EQUAL 0)
  FILE(WRITE ${CMAKE_BINARY_DIR}/source/VERSION \"\${GIT_DESCRIBE_VERSION}\")
ELSE(GIT_DESCRIBE_RESULT EQUAL 0)
  FILE(READ  ${CMAKE_SOURCE_DIR}/VERSION GIT_DESCRIBE_VERSION)
ENDIF(GIT_DESCRIBE_RESULT EQUAL 0)

STRING(REGEX REPLACE \"v([0-9]*)\\\\.([0-9]*)\\\\.(.*)\"
       \"\\\\1.\\\\2.\\\\3\" VERSION \"\${GIT_DESCRIBE_VERSION}\" )

STRING(REGEX REPLACE \"v([0-9]*)\\\\.([0-9]*)\\\\.(.*)\"
       \"\\\\1\" VERSION_MAJOR \"\${GIT_DESCRIBE_VERSION}\")
STRING(REGEX REPLACE \"v([0-9]*)\\\\.([0-9]*)\\\\.(.*)\"
       \"\\\\2\" VERSION_MINOR \"\${GIT_DESCRIBE_VERSION}\")
STRING(REGEX REPLACE \"v([0-9]*)\\\\.([0-9]*)\\\\.(.*)\"
       \"\\\\3\" VERSION_PATCH \"\${GIT_DESCRIBE_VERSION}\")
")

FILE(WRITE ${CMAKE_BINARY_DIR}/FillVersion.cmake
"INCLUDE(Version.cmake)
CONFIGURE_FILE(\${SRC} \${DST} @ONLY)
")
INCLUDE_DIRECTORIES(${CMAKE_BINARY_DIR})
ADD_EXECUTABLE(main main.c)
INSTALL(TARGETS main
  RUNTIME DESTINATION bin
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib
)
ADD_CUSTOM_TARGET(
    version
    ${CMAKE_COMMAND} -D SRC=${CMAKE_BINARY_DIR}/version.h.in
                     -D DST=${CMAKE_BINARY_DIR}/version.h
                     -D WORKING_DIRECTORY=${CMAKE_SOURCE_DIR}
                     -P ${CMAKE_BINARY_DIR}/FillVersion.cmake
)
SET(WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}")
INCLUDE(${CMAKE_BINARY_DIR}/Version.cmake)
SET(CPACK_PACKAGE_VERSION "${VERSION}")
SET(CPACK_PACKAGE_VERSION_MAJOR "${VERSION_MAJOR}")
SET(CPACK_PACKAGE_VERSION_MINOR "${VERSION_MINOR}")
SET(CPACK_PACKAGE_VERSION_PATCH "${VERSION_PATCH}")
set(CPACK_PACKAGE_INSTALL_DIRECTORY "VERSIONHEADER-${VERSIONHEADER_VERSION_FULL}")
SET(CPACK_SOURCE_INSTALLED_DIRECTORIES "${CMAKE_SOURCE_DIR};/;${CMAKE_BINARY_DIR}/source;/")
set(CPACK_SOURCE_IGNORE_FILES
"~$"
".sw[a-z]$"
".DS_Store"
"tags"
"cscope.*"
"^${CMAKE_SOURCE_DIR}/.git.*"
)

INCLUDE(CPack)

IF (UNIX)
  FILE(WRITE ${CMAKE_BINARY_DIR}/Dist.cmake
"MESSAGE(STATUS \"Package Dist running...\")
INCLUDE(CPackSourceConfig.cmake)
EXECUTE_PROCESS(COMMAND cpack -G TGZ --config CPackSourceConfig.cmake
  TIMEOUT 3600
  WORKING_DIRECTORY \${CMAKE_BINARY_DIR})
")
  #
  # Add custom target
  #
  ADD_CUSTOM_TARGET(dist
    COMMAND ${CMAKE_COMMAND} .
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_BINARY_DIR}/Dist.cmake
    )
ENDIF(UNIX)

ADD_DEPENDENCIES(main version)
ADD_DEPENDENCIES(dist version)
