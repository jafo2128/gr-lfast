# Copyright 2011,2012,2016 Free Software Foundation, Inc.
#
# This file is part of GNU Radio
#
# GNU Radio is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; either version 3, or (at your option)
# any later version.
#
# GNU Radio is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with GNU Radio; see the file COPYING.  If not, write to
# the Free Software Foundation, Inc., 51 Franklin Street,
# Boston, MA 02110-1301, USA.

set (CMAKE_CXX_STANDARD 11)

# Reference for detection
# https://gist.github.com/hideo55/5642892

IF(UNIX)
	IF(NOT APPLE)
    	MESSAGE(STATUS "Detected Linux OS")
		EXEC_PROGRAM(cat ARGS "/proc/cpuinfo" OUTPUT_VARIABLE CPUINFO)
	ELSE(NOT APPLE)
    	MESSAGE(STATUS "Detected Apple OS")
		EXEC_PROGRAM("/usr/sbin/sysctl -n machdep.cpu.features" OUTPUT_VARIABLE CPUINFO)
	ENDIF(NOT APPLE)
	
	STRING(REGEX REPLACE "^.*(fma).*$" "fma" FMA_THERE ${CPUINFO})
   STRING(COMPARE EQUAL "fma" "${FMA_THERE}" FMA_TRUE)
   IF (FMA_TRUE)
		MESSAGE(STATUS "Found FMA.  Setting fma compile flag.")
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfma")
   ELSE (FMA_TRUE)
		MESSAGE(STATUS "Not using FMA")
	ENDIF (FMA_TRUE)
ELSE()
    MESSAGE(STATUS "OS is not linux.  Not setting FMA flags.")
ENDIF()

IF(CMAKE_BUILD_TYPE MATCHES DEBUG)
	MESSAGE(STATUS "building for debug...")
ELSE(CMAKE_BUILD_TYPE MATCHES DEBUG)
	MESSAGE(STATUS "Release mode: increasing compiler optimizations...")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O2")
ENDIF(CMAKE_BUILD_TYPE MATCHES DEBUG)

# If you wanted to explicitly force fma you can enable this:
# set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mfma")

# you can also try adding --fast-math to make ALL calculations faster, however
# it gets away from the IEEE standard and can cause issues.  However I don't think
# it will in the code in this library.  However the performance ends up being
# about the same as when fma is enabled.

########################################################################
# Setup library
########################################################################
include(GrPlatform) #define LIB_SUFFIX

include_directories(${Boost_INCLUDE_DIR})
link_directories(${Boost_LIBRARY_DIRS})

list(APPEND lfast_sources
    costas2_impl.cc
    costas4_impl.cc
    agc_fast_impl.cc
    agc_fast_ff_impl.cc
    CC2F2ByteVector_impl.cc
)

set(lfast_sources "${lfast_sources}" PARENT_SCOPE)
if(NOT lfast_sources)
	MESSAGE(STATUS "No C++ sources... skipping lib/")
	return()
endif(NOT lfast_sources)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mfma4")

add_library(gnuradio-lfast SHARED ${lfast_sources})
target_link_libraries(gnuradio-lfast ${Boost_LIBRARIES} ${GNURADIO_ALL_LIBRARIES} boost_thread gnuradio-blocks volk
)
set_target_properties(gnuradio-lfast PROPERTIES DEFINE_SYMBOL "gnuradio_lfast_EXPORTS")

if(APPLE)
    set_target_properties(gnuradio-lfast PROPERTIES
        INSTALL_NAME_DIR "${CMAKE_INSTALL_PREFIX}/lib"
    )
endif(APPLE)

########################################################################
# Install built library files
########################################################################
include(GrMiscUtils)
GR_LIBRARY_FOO(gnuradio-lfast RUNTIME_COMPONENT "lfast_runtime" DEVEL_COMPONENT "lfast_devel")

########################################################################
# Build and register unit test
########################################################################
include(GrTest)

include_directories(${CPPUNIT_INCLUDE_DIRS})

list(APPEND test_lfast_sources
    ${CMAKE_CURRENT_SOURCE_DIR}/test_lfast.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/qa_lfast.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/qa_CC2F2ByteVector.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/qa_agc_fast.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/qa_costas2.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/costas2_impl.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/costas4_impl.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/agc_fast_impl.cc
    ${CMAKE_CURRENT_SOURCE_DIR}/CC2F2ByteVector_impl.cc
)

add_executable(test-lfast ${test_lfast_sources})

target_link_libraries(
  test-lfast
  ${GNURADIO_RUNTIME_LIBRARIES}
  ${Boost_LIBRARIES}
  boost_thread
  ${CPPUNIT_LIBRARIES}
  gnuradio-blocks
  volk
  gnuradio-lfast
)

install(TARGETS test-lfast DESTINATION "${CMAKE_INSTALL_PREFIX}/bin" RUNTIME)


GR_ADD_TEST(test_lfast test-lfast)

########################################################################
# Print summary
########################################################################
message(STATUS "Using install prefix: ${CMAKE_INSTALL_PREFIX}")
message(STATUS "Building for version: ${VERSION} / ${LIBVER}")

