
# Need to use -fno-strict-aliasing when compiling cython code, in order
# to avoid nasty compiler warnings about aliasing.  Cython explicitly
# performs aliasing, in order to emulate python object inheritance.
# See, for example,
# https://groups.google.com/forum/#!topic/cython-users/JV1-KvIUeIg
#
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-strict-aliasing")

INCLUDE_DIRECTORIES(
	${PYTHON_INCLUDE_DIRS}
	${CMAKE_CURRENT_SOURCE_DIR}
	${CMAKE_CURRENT_BINARY_DIR}
)

SET(CYTHON_FLAGS "-f")

# Cythonizes one .pyx file into a .cpp file
# Additional arguments are dependencies
MACRO(CYTHON_ADD_MODULE_PYX name)
	IF(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${name}.pxd)
		SET(DEPENDS ${name}.pyx ${name}.pxd)
	ELSE(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${name}.pxd)
		SET(DEPENDS ${name}.pyx)
	ENDIF(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${name}.pxd)
	# Allow the user to specify dependencies as optional arguments
	SET(DEPENDS ${DEPENDS} ${ARGN})
	ADD_CUSTOM_COMMAND(
		OUTPUT ${name}.cpp
		COMMAND ${CYTHON_EXECUTABLE}
		ARGS ${CYTHON_FLAGS} -I ${PROJECT_BINARY_DIR} -o ${name}.cpp
				--cplus ${CMAKE_CURRENT_SOURCE_DIR}/${name}.pyx
		DEPENDS ${DEPENDS}
		COMMENT "Cythonizing ${name}.pyx")
ENDMACRO(CYTHON_ADD_MODULE_PYX)

# Use this as a guide:
# https://github.com/OpenKinect/libfreenect/blob/master/wrappers/python/CMakeLists.txt

###################### logger ##########################################
CYTHON_ADD_MODULE_PYX(logger)
list(APPEND ADDITIONAL_MAKE_CLEAN_FILES "logger.cpp"
	"../../util/Logger.h")

ADD_LIBRARY(logger_cython
	logger.cpp
)

TARGET_LINK_LIBRARIES(logger_cython
	${COGUTIL_LIBRARY}
	${PYTHON_LIBRARIES}
)

SET_TARGET_PROPERTIES(logger_cython PROPERTIES
	PREFIX ""
	OUTPUT_NAME logger)

###################### atomspace ####################################
CYTHON_ADD_MODULE_PYX(atomspace
	"atom.pyx" "classserver.pyx" "truth_value.pyx"
	"atomspace_details.pyx" opencog_atom_types
	"../../truthvalue/TruthValue.h" "../../truthvalue/SimpleTruthValue.h"
	"../../atoms/base/ClassServer.h" "../../atoms/base/Handle.h"
	"../../atomspace/AtomSpace.h"
)

# list(APPEND ADDITIONAL_MAKE_CLEAN_FILES "atomspace.cpp")
list(APPEND ADDITIONAL_MAKE_CLEAN_FILES "atomspace_api.h")

# opencog.atomspace Python bindings
ADD_LIBRARY(atomspace_cython
	Cast.cc
	atomspace.cpp
)

# XXX FIXME Cython should not static link the atom-types, but
# but should instead dynamic-load them, as needed.
ADD_DEPENDENCIES(atomspace_cython opencog_atom_types)

TARGET_LINK_LIBRARIES(atomspace_cython
	-Wl,--no-as-needed
	attentionbank
	atomspace
	atomutils
	clearbox
	atomspaceutils
	truthvalue
	atomcore
	atomspace
	${COGUTIL_LIBRARY}
	${PYTHON_LIBRARIES}
	${Boost_THREAD_LIBRARY}
	${Boost_SYSTEM_LIBRARY}
)

SET_TARGET_PROPERTIES(atomspace_cython PROPERTIES
	PREFIX ""
	OUTPUT_NAME atomspace)

# The cogserver needs the atomspace pxd file to get the Handle, Atom and
# AtomSpace defintions during compilation. So this must be installed.
INSTALL (FILES
	__init__.py
	atomspace.pxd
	Cast.h
	DESTINATION "include/opencog/cython/opencog"
)

############################## type constructors #####################
CYTHON_ADD_MODULE_PYX(type_constructors
	"atomspace.pxd" opencog_atom_types )

list(APPEND ADDITIONAL_MAKE_CLEAN_FILES "type_constructors.cpp")

ADD_LIBRARY(type_constructors
	type_constructors.cpp
)

TARGET_LINK_LIBRARIES(type_constructors
	atomspace
	${PYTHON_LIBRARIES}
)

SET_TARGET_PROPERTIES(type_constructors PROPERTIES
	PREFIX ""
	OUTPUT_NAME type_constructors)


############################## utilities #####################
CYTHON_ADD_MODULE_PYX(utilities
	"atomspace.pxd" opencog_atom_types )

list(APPEND ADDITIONAL_MAKE_CLEAN_FILES "utilities.cpp")

ADD_LIBRARY(utilities_cython
	Utilities.cc
	utilities.cpp
)

TARGET_LINK_LIBRARIES(utilities_cython
	PythonEval
	clearbox
	atomspace
	type_constructors
	${COGUTIL_LIBRARY}
	${PYTHON_LIBRARIES}
	${Boost_THREAD_LIBRARY}
	${Boost_SYSTEM_LIBRARY}
)

SET_TARGET_PROPERTIES(utilities_cython PROPERTIES
	PREFIX ""
	OUTPUT_NAME utilities)

############################## bindlink #####################
CYTHON_ADD_MODULE_PYX(bindlink
	"atomspace.pxd" opencog_atom_types
	"../../query/BindLinkAPI.h")

list(APPEND ADDITIONAL_MAKE_CLEAN_FILES "bindlink.cpp")

ADD_LIBRARY(bindlink_cython
	BindlinkStub.cc
	bindlink.cpp
)

TARGET_LINK_LIBRARIES(bindlink_cython
	atomspace_cython
	query
	execution
	${PYTHON_LIBRARIES}
	${Boost_THREAD_LIBRARY}
	${Boost_SYSTEM_LIBRARY}
)

SET_TARGET_PROPERTIES(bindlink_cython PROPERTIES
	PREFIX ""
	OUTPUT_NAME bindlink)

# XXX FIXME -- there must surely be a more elegant way
# of specifying the correct python install path ...
INSTALL (TARGETS
	atomspace_cython
	bindlink_cython
	logger_cython
	type_constructors
	utilities_cython
	DESTINATION "lib${LIB_DIR_SUFFIX}/python2.7/dist-packages/opencog")

IF (HAVE_GUILE)
	############################## scheme wrapper #####################
	CYTHON_ADD_MODULE_PYX(scheme_wrapper
		"PyScheme.h" "atomspace.pxd" opencog_atom_types
		"../load-file.h")

	list(APPEND ADDITIONAL_MAKE_CLEAN_FILES "scheme_wrapper.cpp")

	ADD_LIBRARY(scheme_wrapper
		PyScheme
		scheme_wrapper.cpp
	)

	TARGET_LINK_LIBRARIES(scheme_wrapper
		atomspace_cython
		query
		execution
		smob
		${PYTHON_LIBRARIES}
	)
	
	SET_TARGET_PROPERTIES(scheme_wrapper PROPERTIES
		PREFIX ""
		OUTPUT_NAME scheme_wrapper)

	INSTALL (TARGETS
		scheme_wrapper
		DESTINATION "lib${LIB_DIR_SUFFIX}/python2.7/dist-packages/opencog")
ENDIF (HAVE_GUILE)


INSTALL (FILES
	__init__.py
	DESTINATION "lib${LIB_DIR_SUFFIX}/python2.7/dist-packages/opencog")
