cmake_minimum_required(VERSION 3.20)

project(i18n++
  VERSION 0.0
  DESCRIPTION "gettext for C++"
  LANGUAGES C CXX)

set(CMAKE_CXX_EXTENSIONS off)

option(BUILD_TESTING "Build tests" ON)
option(BUILD_PLUGIN "Build Clang plugin" ON)
option(BUILD_MERGE "Build i18n-merge-pot tool" ON)

include(CTest)

add_subdirectory(external)
if(BUILD_MERGE)
  add_subdirectory(merge)
endif()

if(BUILD_PLUGIN)
  add_subdirectory(clang)
endif()

find_package(Intl REQUIRED)

add_library(i18n-lib INTERFACE)
target_link_libraries(i18n-lib INTERFACE Intl::Intl)
target_include_directories(i18n-lib INTERFACE
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>"
  "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>")
if(BUILD_PLUGIN)
  target_compile_options(i18n-lib INTERFACE "$<IF:$<CXX_COMPILER_ID:Clang>,-fplugin=$<TARGET_FILE:$<INSTALL_INTERFACE:i18n::>plugin>,>")
endif()
target_compile_features(i18n-lib INTERFACE cxx_std_20)
target_compile_definitions(i18n-lib INTERFACE "$<IF:$<PLATFORM_ID:Darwin>,_INTL_REDIRECT_MACROS,>")

add_custom_target(i18n_internal)
if(BUILD_PLUGIN)
  add_dependencies(i18n_internal plugin)
endif()
add_dependencies(i18n-lib i18n_internal)


add_library(i18n::i18n-lib ALIAS i18n-lib)
if(BUILD_MERGE)
  add_executable(i18n::i18n-merge-pot ALIAS i18n-merge-pot)
endif()
if(BUILD_PLUGIN)
  add_executable(i18n::i18n-extract ALIAS i18n-extract)
  include ( "${CMAKE_CURRENT_LIST_DIR}/i18n++Use.cmake" )
endif()

if(BUILD_TESTING)
  add_subdirectory(tests)
endif()

include(GNUInstallDirs)

install(TARGETS i18n-lib EXPORT i18n++Targets)
install(DIRECTORY include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR} FILES_MATCHING PATTERN "*.hpp")
install(EXPORT i18n++Targets NAMESPACE i18n:: FILE i18n++Targets.cmake DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/i18n++)

include(CMakePackageConfigHelpers)

configure_package_config_file(Config.cmake.in i18n++Config.cmake
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/i18n++)
write_basic_package_version_file(
  i18n++ConfigVersion.cmake
  VERSION "${i18n++_VERSION_MAJOR}.${i18n++_VERSION_MINOR}"
  COMPATIBILITY AnyNewerVersion
)

INSTALL(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/i18n++Config.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/i18n++ConfigVersion.cmake
  i18n++Use.cmake
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/i18n++)

include(CPack)
