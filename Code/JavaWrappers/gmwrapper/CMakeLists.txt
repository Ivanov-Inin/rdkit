project (jGraphMolJava Java)

find_package(Java REQUIRED)
find_package(JNI REQUIRED)

# Make sure we can access the junit.jar -- needed for compiling/running tests
find_file(JUNIT_JAR junit.jar ${RDKit_JavaLibDir})
if (NOT JUNIT_JAR)
	MESSAGE ("junit.jar is not found in ${RDKit_JavaLibDir}.  Please add it to this directory and rerun cmake.")
	MESSAGE(FATAL_ERROR "Cannot find required JUnit library (junit.jar)")
endif (NOT JUNIT_JAR)

# Likewise for javadoc
find_program(JAVADOC_EXE javadoc PATH _JAVA_PATHS _JAVA_HINTS /usr/bin /usr/local/bin)
if (NOT JAVADOC_EXE)
	MESSAGE ("javadoc (executable) is not found. Please add it to PATH and rerun cmake.")
	MESSAGE(FATAL_ERROR "Cannot find required javadoc executable (javadoc)")
endif (NOT JAVADOC_EXE)

INCLUDE_DIRECTORIES(${JNI_INCLUDE_DIRS})

SET_SOURCE_FILES_PROPERTIES(GraphMolJava.i PROPERTIES CPLUSPLUS ON )


FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/../*.java COPY_SOURCE)
FILE(TO_NATIVE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/src/org/RDKit COPY_DEST)

# Setup a few variables for environment-specific things
if(WIN32)
  ADD_DEFINITIONS("/W3 /wd4716")
  SET(PATH_SEP ";")
  SET(COPY_CMD xcopy ${COPY_SOURCE} ${COPY_DEST} /Y /I)
else()
  SET(PATH_SEP ":")
  SET(COPY_CMD cp -p ${COPY_SOURCE} ${COPY_DEST})
endif()

# Coax SWIG into playing nicely with Apple environments
if(APPLE)
  SET(CMAKE_SIZEOF_VOID_P 4)
endif(APPLE)

if(CMAKE_SIZEOF_VOID_P MATCHES 4)
  SET(CMAKE_SWIG_FLAGS -package "org.RDKit")
else()
  SET(CMAKE_SWIG_FLAGS -package "org.RDKit" "-DSWIGWORDSIZE64")
endif()
SET(CMAKE_SWIG_OUTDIR ${CMAKE_CURRENT_SOURCE_DIR}/src/org/RDKit )

# enable this line to build the ErrorGenerator class for testing handling of C++ errors in the JNI layer
#SET(CMAKE_SWIG_FLAGS ${CMAKE_SWIG_FLAGS} "-D INCLUDE_ERROR_GENERATOR" )

SWIG_ADD_MODULE(GraphMolWrap "java" GraphMolJava.i)

SET(SWIG_MODULE_GraphMolWrap_EXTRA_DEPS ../*.i)


SWIG_LINK_LIBRARIES(GraphMolWrap ${RDKit_Wrapper_Libs} )

# code adapted from the wrapper code for
# GDCM: http://gdcm.svn.sf.net/viewvc/gdcm/trunk/Wrapping/Java/CMakeLists.txt?view=markup
ADD_CUSTOM_COMMAND(
  OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar

  ## Add in our own Java sources
  #COMMAND ${COPY_CMD}
  ## 1. run this custom command only after swig has been run.
  COMMAND ${JAVA_COMPILE} -d build "src/org/RDKit/*.java"
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  ## 2. now that the *.class have been generated construct the jar file.
  COMMAND ${JAVA_ARCHIVE} cf
    ${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar -C ${CMAKE_CURRENT_SOURCE_DIR}/build
    org/RDKit
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  #DEPENDS GraphMolWrap
  DEPENDS "${swig_generated_file_fullname}"
  COMMENT "javac *.java; jar cf -> .jar"

  OUTPUT ${CMAKE_CURRENT_SOURCE_DIR}/org.RDKitDoc.jar
  ## Run Javadoc against newly created .java files to create docs
  COMMAND ${JAVADOC_EXE} -tag notes -tag example -d ${CMAKE_CURRENT_SOURCE_DIR}/doc -sourcepath ${CMAKE_CURRENT_SOURCE_DIR}/src org.RDKit
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  ## Put the doc files into their own separate archive.
  COMMAND ${JAVA_ARCHIVE} cf 
    ${CMAKE_CURRENT_SOURCE_DIR}/org.RDKitDoc.jar
    doc
    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  #DEPENDS GraphMolWrap
  DEPENDS "${swig_generated_file_fullname}"
  COMMENT "jar cf org.RDKitDoc.jar -d doc"
)
ADD_CUSTOM_TARGET(GraphMolWrapJar ALL
  DEPENDS GraphMolWrap ${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar
  COMMENT "building jar"
)

## Tests -- note that building/rebuilding them keys on the file WrapperTests.class

SET(CMAKE_JAVA_TEST_OUTDIR ${CMAKE_CURRENT_SOURCE_DIR}/build-test )
ADD_CUSTOM_COMMAND(
  OUTPUT ${CMAKE_JAVA_TEST_OUTDIR}/org/RDKit/WrapperTests.class
  COMMAND ${JAVA_COMPILE} -d ${CMAKE_JAVA_TEST_OUTDIR} -cp "${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar${PATH_SEP}${JUNIT_JAR}" "src-test/org/RDKit/*.java"
  WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
  DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar 
)
ADD_CUSTOM_TARGET(BuildJavaWrapperTests ALL
  DEPENDS ${CMAKE_JAVA_TEST_OUTDIR}/org/RDKit/WrapperTests.class GraphMolWrap 
  COMMENT "building test classes"
)
ADD_CUSTOM_TARGET(RunJavaWrapperTests 
  DEPENDS BuildJavaWrapperTests
  COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/runUnitTests.sh
)

ADD_TEST(JavaAromaticTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar"
	org.RDKit.AromaticTests)
ADD_TEST(JavaAtomPairsTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.AtomPairsTests)
ADD_TEST(JavaBasicMoleculeTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.BasicMoleculeTests)
ADD_TEST(JavaBasicMolecule2Tests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.BasicMolecule2Tests)
ADD_TEST(JavaChemAtomTests
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.ChemAtomTests)
ADD_TEST(JavaChemBondTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.ChemBondTests)
ADD_TEST(JavaChemReactionTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar"
	org.RDKit.ChemReactionTests)
ADD_TEST(JavaChemSmartsTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.ChemSmartsTests)
ADD_TEST(JavaChemTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.ChemTests)
ADD_TEST(JavaChemv2Tests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.Chemv2Tests)
ADD_TEST(JavaConformerTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.ConformerTests)
ADD_TEST(JavaDescriptorTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.DescriptorTests)
ADD_TEST(JavaDistanceGeometryTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.DistanceGeometryTests)
ADD_TEST(JavaErrorHandlingTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.ErrorHandlingTests)
ADD_TEST(JavaFingerprintsTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.FingerprintsTests)
ADD_TEST(JavaForceFieldsTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.ForceFieldsTests)
ADD_TEST(JavaHManipulationsTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.HManipulationsTests)
ADD_TEST(JavaLipinskiTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.LipinskiTests)
ADD_TEST(JavaPicklingTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.PicklingTests)
ADD_TEST(JavaSmilesCreationTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.SmilesCreationTests)
ADD_TEST(JavaSmilesDetailsTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.SmilesDetailsTests)
ADD_TEST(JavaSmilesTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.SmilesTests)
ADD_TEST(JavaSuppliersTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.SuppliersTests)
ADD_TEST(JavaWrapperTests 
    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
	org.RDKit.WrapperTests)
       
#ADD_TEST(JavaMemoryTests 
#    java -Djava.library.path=${CMAKE_CURRENT_SOURCE_DIR} 
#	-cp "${JUNIT_JAR}${PATH_SEP}${CMAKE_JAVA_TEST_OUTDIR}${PATH_SEP}${CMAKE_CURRENT_SOURCE_DIR}/org.RDKit.jar" 
#	org.RDKit.MemoryTests)


INSTALL(TARGETS GraphMolWrap 
              DESTINATION ${CMAKE_CURRENT_SOURCE_DIR} )