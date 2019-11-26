# Original work Copyright (c) 2017, Mate Soos
# Modified work Copyright (c) 2017-2019, Pierre Vignet
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.

function(get_link_dependencies _target _listvar)
    # This function gets all dependencies of the given target.
    # It avoids to respecify the source files already mentioned for a given target
    # in other CMakeLists.
    set(_worklist ${${_listvar}})
    if (TARGET ${_target})
        list(APPEND _worklist ${_target})
        get_property(_dependencies GLOBAL PROPERTY GlobalTargetDepends${_target})
        foreach(_dependency IN LISTS _dependencies)
            list(FIND _worklist ${_dependency} _idx)
            if (${_idx} EQUAL -1)
                get_link_dependencies(${_dependency} _worklist)
            endif()
        endforeach()
        set(${_listvar} "${_worklist}" PARENT_SCOPE)
    endif()
endfunction()


## Copy and configure sources of libraries in the current folder to prepare the setuptools source package
# Why do we need to copy all sources here before the packaging ?
# Setuptools does not build "out of sources". In order to fix this, all Python and C sources
# have to be copied in the python source tree in CMAKE_CURRENT_BINARY_DIR.

# Insert version number and build environment:
configure_file("${PROJECT_SOURCE_DIR}/src/GitSHA1.cpp.in"
    "${CMAKE_CURRENT_BINARY_DIR}/src/GitSHA1.cpp" @ONLY)
# Insert version number:
configure_file("${PROJECT_SOURCE_DIR}/src/cryptominisat.h.in"
    "${CMAKE_CURRENT_BINARY_DIR}/src/cryptominisat5/cryptominisat.h" @ONLY)
# Do nothing but we need these headers:
configure_file("${PROJECT_SOURCE_DIR}/src/cryptominisat_c.h.in"
    "${CMAKE_CURRENT_BINARY_DIR}/src/cryptominisat5/cryptominisat_c.h" @ONLY)
configure_file("${PROJECT_SOURCE_DIR}/src/solvertypesmini.h.in"
    "${CMAKE_CURRENT_BINARY_DIR}/src/cryptominisat5/solvertypesmini.h" @ONLY)
# Copy pycryptosat itself
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/src/pycryptosat.cpp"
    "${CMAKE_CURRENT_BINARY_DIR}/src/pycryptosat.cpp" COPYONLY)

# Copy headers
file(GLOB HEADERS
  "${PROJECT_SOURCE_DIR}/src/*.h"
)
file(COPY ${HEADERS} DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/src/")

# Get source files for the target libcryptominisat5 and copy them
get_link_dependencies(libcryptominisat5 _deps)
foreach(_dep IN LISTS _deps)
    get_target_property(_srcs ${_dep} SOURCES)
    foreach(_src IN LISTS _srcs)
        # Remove .../build/cmsat5-src/GitSHA1.cpp that doesn't exists in the ${PROJECT_SOURCE_DIR}
        # TODO: could just add an else() statement to copy it directly from the path given in ${_src}
        if(EXISTS "${PROJECT_SOURCE_DIR}/src/${_src}")
            configure_file(
                "${PROJECT_SOURCE_DIR}/src/${_src}"
                "${CMAKE_CURRENT_BINARY_DIR}/src/" COPYONLY)
        endif()
    endforeach()
endforeach()


## Get all sources files and format them into a string that will be inserted in the setup.py
## See PYTHON_STRIPPED_SOURCES
file(GLOB SOURCES
  "${CMAKE_CURRENT_BINARY_DIR}/src/*.cpp"
)
# Remove the current path (not needed for setup.py) for each file
set(PYTHON_STRIPPED_SOURCES)
foreach(file ${SOURCES})
    string(REPLACE "${CMAKE_CURRENT_BINARY_DIR}/" "" file ${file})
    set(PYTHON_STRIPPED_SOURCES "${PYTHON_STRIPPED_SOURCES}\"${file}\", ")
endforeach()

# Configure setup.py with project version and sources
set(SETUP_PY    "${CMAKE_CURRENT_BINARY_DIR}/setup.py")
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/setup.py.in" ${SETUP_PY})


## Copy additional files that will be integrated in the setuptools source package
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/README.rst" "${CMAKE_CURRENT_BINARY_DIR}/README.rst" COPYONLY)
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/LICENSE" "${CMAKE_CURRENT_BINARY_DIR}/LICENSE" COPYONLY)
configure_file("${CMAKE_CURRENT_SOURCE_DIR}/MANIFEST.in" "${CMAKE_CURRENT_BINARY_DIR}/MANIFEST.in" COPYONLY)
file(COPY "${CMAKE_CURRENT_SOURCE_DIR}/tests" DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/")


## pycryptosat targets

# Build
# timestamp: the output from python setup.py build is not a good candidate to detect
# if build has to run, because it is placed in architecture- and platform-specific
# directories like lib.linux-x86_64-2.6.
# A timestamp file is generated each time the source dependencies changes.
set(OUTPUT      "${CMAKE_CURRENT_BINARY_DIR}/build/timestamp")
add_custom_command(OUTPUT ${OUTPUT}
                   COMMAND ${PYTHON_EXECUTABLE}
                   ARGS ${SETUP_PY} build
                   DEPENDS ${SETUP_PY}
                   COMMAND ${CMAKE_COMMAND} -E touch ${OUTPUT})
add_custom_target(pycryptosat_pypi
                  DEPENDS ${OUTPUT}
                  COMMENT "Building pycryptosat for pypi...")
# Install in dev mode
# This procedure makes symlinks from the actual Python environment to the current module
# and allows to modify and test files during the development process without reinstalling the project.
# Moreover, this install/uninstall method is much proper than the old and deprecated one:
# "python setup.py install"
# => No file is left behind here.
add_custom_target(pycryptosat_pypi_dev_install
                  COMMAND ${PYTHON_EXECUTABLE} ${SETUP_PY} develop
                  COMMENT "Install pycryptosat for pypi in develop mode...")
# Uninstall in dev mode
add_custom_target(pycryptosat_pypi_dev_uninstall
                  COMMAND ${PYTHON_EXECUTABLE} ${SETUP_PY} develop --uninstall
                  COMMENT "Uninstall pycryptosat for pypi previously installed in develop mode...")
# Make a source package
# This package can be directly uploaded to PyPI
add_custom_target(pycryptosat_pypi_dist_package
                  COMMAND ${PYTHON_EXECUTABLE} ${SETUP_PY} sdist
                  COMMENT "Build pycryptosat for pypi source package...")
# Make a binary package
# This binary package (wheel) can be uploaded on PyPI.
# It avoids users having to compile the source package.
#
# PS: Currently (oct 2017), C++14 can not be compiled in the environment recommended for the builds
# of the wheels (CentOS 5.11 https://www.python.org/dev/peps/pep-0513/#id40);
# consequently the binaries compiled outside this environment can not have the tag manylinux1
# allowing them to be uploaded on pypi. The trick is to compile on a more recent distribution
# (but almost outdated like Debian Jessie for example) in order to link the library to a version
# of the lib C compatible with the majority of the recent distributions (by backward compatibility);
# then simply replace the tag "linux_x86_64" with "manylinux1_x86_64" in the name of the generated whl file.
add_custom_target(pycryptosat_pypi_bdist_wheel_package
                  COMMAND ${PYTHON_EXECUTABLE} ${SETUP_PY} bdist_wheel
                  COMMENT "Build pycryptosat for pypi binary package...")


if (ENABLE_TESTING AND NOT APPLE)
    add_custom_target(pycryptosat_pypi_test
             COMMAND ${PYTHON_EXECUTABLE} ${SETUP_PY} test)
endif()
