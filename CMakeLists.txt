cmake_minimum_required(VERSION 3.10)
project(waves VERSION 0.0.1 LANGUAGES CXX)

find_program(CCACHE ccache)
if(CCACHE)
  set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE})
endif()

##
## Set compiler flags before running conan
##

# Link this 'library' to use the standard warnings
add_library(project_warnings INTERFACE)

# Link this 'library' to set the c++ standard / compile-time options requested
add_library(project_options INTERFACE)

target_compile_features(project_options INTERFACE cxx_std_17)
target_include_directories(project_options INTERFACE include external)

if(MSVC)
  target_compile_options(project_warnings INTERFACE /W4 /WX "/permissive-")
else()
  option(ONLY_COVERAGE "Build only tests necessary for coverage" FALSE)
  option(LIBCPP "Build with libc++" FALSE)
  option(ENABLE_COVERAGE "Enable coverage reporting for gcc/clang" TRUE)

  if(ONLY_COVERAGE OR ENABLE_COVERAGE)
    target_compile_options(project_options INTERFACE --coverage -O0 -g)
    target_link_libraries(project_options INTERFACE --coverage )
  endif()

  option(ENABLE_ASAN "Enable address sanitizer" FALSE)

  if(ENABLE_ASAN)
    target_compile_options(project_options INTERFACE -fsanitize=address)
    target_link_libraries(project_options INTERFACE -fsanitize=address)
  endif()

  target_compile_options(project_warnings
                         INTERFACE -Wall
                                   -Werror # treat all warnings as errors
                                   -Wextra # reasonable and standard
                                   -Wshadow # warn the user if a variable declaration shadows one
                                            # from a parent context
                                   -Wnon-virtual-dtor # warn the user if a class with virtual
                                                      # functions has a non-virtual destructor. This
                                                      # helps catch hard to track down memory errors
                                   -Wold-style-cast # warn for c-style casts
                                   -Wcast-align # warn for potential performance problem casts
                                   -Wunused # warn on anything being unused
                                   -Woverloaded-virtual # warn if you overload (not override) a
                                                        # virtual function
                                   -Wpedantic # warn if non-standard C++ is used
                                   -Wconversion # warn on type conversions that may lose data
                                   -Wsign-conversion # warn on sign conversions
                                   -Wnull-dereference # warn if a null dereference is detected
                                   -Wdouble-promotion # warn if float is implicit promoted to double
                                   -Wformat=2 # warn on security issues around functions that format
                                              # output (ie printf)
                         )

  if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    target_compile_options(project_warnings
                           INTERFACE -Wmisleading-indentation # warn if identation implies blocks
                                                              # where blocks do not exist
                                     -Wduplicated-cond # warn if if / else chain has duplicated
                                                       # conditions
                                     -Wduplicated-branches # warn if if / else branches have
                                                           # duplicated code
                                     -Wlogical-op # warn about logical operations being used where
                                                  # bitwise were probably wanted
                                     -Wuseless-cast # warn if you perform a cast to the same type
                           )
    target_link_libraries(project_options INTERFACE stdc++fs)
  else()
    if(LIBCPP)
      target_compile_options(project_options INTERFACE -stdlib=libc++)
    else()
      target_link_libraries(project_options INTERFACE stdc++fs)
    endif()

  endif()
endif()

##
## Load Conan
##

if(NOT EXISTS "${CMAKE_BINARY_DIR}/conan.cmake")
  message(STATUS "Downloading conan.cmake from https://github.com/conan-io/cmake-conan")
  file(DOWNLOAD "https://raw.githubusercontent.com/conan-io/cmake-conan/v0.12/conan.cmake"
                "${CMAKE_BINARY_DIR}/conan.cmake")
endif()


include(${CMAKE_BINARY_DIR}/conan.cmake)
conan_check(REQUIRED)
conan_cmake_run(CONANFILE conanfile.txt BASIC_SETUP CONAN_COMMAND ${CONAN_CMD} CMAKE_TARGETS BUILD missing)
set(CMAKE_MODULE_PATH /usr/share/SFML/cmake/Modules)

if (CONAN_CATCH2_ROOT_DEBUG)
  include(${CONAN_CATCH2_ROOT_DEBUG}/lib/cmake/Catch2/Catch.cmake)
else ()
  include(${CONAN_CATCH2_ROOT}/lib/cmake/Catch2/Catch.cmake)
endif ()

###########
### Options
###########

option(BUILD_SHARED_LIBS "Enable compilation of shared libraries" FALSE)
option(ENABLE_CLANG_TIDY "Enable testing with clang-tidy" FALSE)
option(ENABLE_CPPCHECK "Enable testing with cppcheck" FALSE)

if(ENABLE_CPPCHECK)
  find_program(CPPCHECK cppcheck)
  if(CPPCHECK)
    set(CMAKE_CXX_CPPCHECK ${CPPCHECK} --suppress=syntaxError --enable=all --inconclusive)
  else()
    message(SEND_ERROR "cppcheck requested but executable not found")
  endif()
endif()

if(ENABLE_CLANG_TIDY)
  find_program(CLANGTIDY clang-tidy)
  if(CLANGTIDY)
    set(CMAKE_CXX_CLANG_TIDY ${CLANGTIDY})
  else()
    message(SEND_ERROR "clang-tidy requested but executable not found")
  endif()
endif()


##################
## Project Setup
##################
enable_testing()

add_executable(basic_tests tests/test_ci.cpp)
target_link_libraries(basic_tests PRIVATE project_options project_warnings CONAN_PKG::catch2 CONAN_PKG::glm)
catch_discover_tests(basic_tests TEST_PREFIX "basic.")

if(NOT ONLY_COVERAGE)
  add_executable(waves src/main.cpp)
  target_link_libraries(waves 
                        PRIVATE project_options 
                                project_warnings 
                                CONAN_PKG::glm)
                                
  #target_include_directories(waves SYSTEM PRIVATE external)
endif()
