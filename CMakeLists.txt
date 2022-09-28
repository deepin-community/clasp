cmake_minimum_required(VERSION 3.1)
project(CLASP VERSION 3.3.5 LANGUAGES CXX)
# Enable folders in IDEs like Visual Studio
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
if (POLICY CMP0063)
	cmake_policy(SET CMP0063 NEW)
endif()
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	message(STATUS "No build type selected - using 'Release'")
	set(CMAKE_BUILD_TYPE "Release")
endif()
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

include(GNUInstallDirs)

# Configuration options
option(CLASP_BUILD_APP          "whether or not to build the clasp application" ON)
option(CLASP_BUILD_STATIC       "whether or not to link statically (if supported)" OFF)
option(CLASP_BUILD_TESTS        "whether or not to build clasp unit tests" OFF)
option(CLASP_BUILD_EXAMPLES     "whether or not to build examples" OFF)
option(CLASP_BUILD_WITH_THREADS "whether or not to build clasp with threading support (requires C++11)" ON)
option(CLASP_INSTALL_LIB        "whether or not to install libclasp" OFF)

if (NOT MSVC)
	if (NOT CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
		set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
	endif()
	if (NOT CMAKE_LIBRARY_OUTPUT_DIRECTORY)
		set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
	endif()
	if (NOT CMAKE_RUNTIME_OUTPUT_DIRECTORY)
		set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
	endif()
else()
	set(VC_RELEASE_LINK_OPTIONS /LTCG)
	SET(CMAKE_EXE_LINKER_FLAGS_RELEASE    "${CMAKE_EXE_LINKER_FLAGS_RELEASE}    ${VC_RELEASE_LINK_OPTIONS}")
	SET(CMAKE_MODULE_LINKER_FLAGS_RELEASE "${CMAKE_MODULE_LINKER_FLAGS_RELEASE} ${VC_RELEASE_LINK_OPTIONS}")
	SET(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_SHARED_LINKER_FLAGS_RELEASE} ${VC_RELEASE_LINK_OPTIONS}")
	SET(CMAKE_STATIC_LINKER_FLAGS_RELEASE "${CMAKE_STATIC_LINKER_FLAGS_RELEASE} ${VC_RELEASE_LINK_OPTIONS}")
	if (CLASP_BUILD_STATIC)
		# force static runtime
		string(REGEX REPLACE "/MD" "/MT" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
	endif()
endif()
set(clasp_include_dest "clasp-${CLASP_VERSION}")
set(clasp_library_dest "clasp-${CLASP_VERSION}")
set(cmake_dest         "clasp-${CLASP_VERSION}/cmake")

if (CLASP_INSTALL_LIB AND NOT CMAKE_INSTALL_LIBDIR)
	message(STATUS "LIBDIR no set - using lib")
	set(CMAKE_INSTALL_LIBDIR lib)
endif()


# C++11 is required for building with threads
if (CLASP_BUILD_WITH_THREADS)
	set(CMAKE_CXX_STANDARD 11)
	set(CMAKE_CXX_STANDARD_REQUIRED ON)
	set(CMAKE_CXX_EXTENSIONS ON)
	# some versions of findThreads will fail if C is not enabled
	enable_language(C)
	find_package(Threads REQUIRED)

	# Add libatomic if necessary
	if (CMAKE_CXX_COMPILER_ID MATCHES "Clang" AND CMAKE_USE_PTHREADS_INIT)
		include (CheckCXXSourceCompiles)
		set (OLD_CMAKE_REQUIRED_FLAGS ${CMAKE_REQUIRED_FLAGS})
		set (OLD_CMAKE_REQUIRED_LIBRARIES ${CMAKE_REQUIRED_LIBRARIES})
		list(APPEND CMAKE_REQUIRED_FLAGS "-std=c++11")
		list(APPEND CMAKE_REQUIRED_LIBRARIES Threads::Threads)
		check_cxx_source_compiles("
#include <atomic>
#include <cstdint>
std::atomic<uint64_t> x (0);
int main() {
	uint64_t i = x.load(std::memory_order_relaxed);
	return 0;
}
" CLASP_HAS_WORKING_LIBATOMIC)
		set (CMAKE_REQUIRED_FLAGS ${OLD_CMAKE_REQUIRED_FLAGS})
		set (CMAKE_REQUIRED_LIBRARIES ${OLD_CMAKE_REQUIRED_LIBRARIES})
		if (NOT CLASP_HAS_WORKING_LIBATOMIC)
			check_library_exists(atomic __atomic_fetch_add_4 "" CLASP_HAS_LIBATOMIC)
			if (CLASP_HAS_LIBATOMIC)
				set_property(TARGET Threads::Threads APPEND PROPERTY INTERFACE_LINK_LIBRARIES "atomic")
			endif()
		endif()
	endif()
endif()

# Check for and optionally build external dependency
find_package(potassco 1.0 QUIET CONFIG)
if (NOT potassco_FOUND)
	if (NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/libpotassco/CMakeLists.txt)
		message(STATUS "Potassco is not installed - fetching submodule")
		execute_process(COMMAND git submodule update --init WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} OUTPUT_QUIET)
	else()
		message(STATUS "Potassco is not installed - using local copy")
	endif()
	set(LIB_POTASSCO_BUILD_APP   ${CLASP_BUILD_APP}   CACHE BOOL "")
	set(LIB_POTASSCO_INSTALL_LIB ${CLASP_INSTALL_LIB} CACHE BOOL "")
	add_subdirectory(libpotassco)
endif()

# Build clasp library
add_subdirectory(src)

# Build optional targets
if(CLASP_BUILD_TESTS)
	enable_testing()
	add_subdirectory(tests)
endif()
# optional doc target
find_package(Doxygen)
if(DOXYGEN_FOUND)
	set(doxyfile "${CMAKE_CURRENT_SOURCE_DIR}/doc/api/clasp.doxy")
	add_custom_target(doc_clasp
		COMMAND ${DOXYGEN_EXECUTABLE} ${doxyfile}
		WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/doc/api"
		COMMENT "Generating documentation..."
		VERBATIM)
	set_target_properties(doc_clasp PROPERTIES FOLDER doc)
endif()

if(CLASP_BUILD_APP)
	add_subdirectory(app)
endif()

if(CLASP_BUILD_EXAMPLES)
	add_subdirectory(examples)
endif()

# Export
configure_file(cmake/clasp-config-version.cmake.in
	${CMAKE_CURRENT_BINARY_DIR}/clasp-config-version.cmake
	@ONLY)

configure_file(cmake/clasp-config.cmake.in
	${CMAKE_CURRENT_BINARY_DIR}/clasp-config.cmake
	@ONLY)

if (CLASP_INSTALL_LIB)
	install(FILES ${CMAKE_CURRENT_BINARY_DIR}/clasp-config.cmake ${CMAKE_CURRENT_BINARY_DIR}/clasp-config-version.cmake
	  DESTINATION "${CMAKE_INSTALL_LIBDIR}/${cmake_dest}")
	install(EXPORT clasp DESTINATION "${CMAKE_INSTALL_LIBDIR}/${cmake_dest}")
endif()
