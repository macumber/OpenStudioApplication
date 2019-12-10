if ("${CMAKE_CXX_COMPILER_ID}" MATCHES "^(Apple)?Clang$")
  # using Clang
  set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -undefined dynamic_lookup")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  # using GCC
  #set(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} --unresolved-symbols=ignore-all")
  set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} --unresolved-symbols=ignore-all")
endif()

if (UNIX)
  # Disable register warnings
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-error=register -Wno-register")
endif()

include_directories(${CMAKE_CURRENT_BINARY_DIR} ${PROJECT_BINARY_DIR} ${PROJECT_SOURCE_DIR} ${CMAKE_CURRENT_SOURCE_DIR} )

add_library(openstudio_modeleditor_rb MODULE
  RubyAPI.hpp
  openstudio_modeleditor_rb.cpp
)
target_include_directories(openstudio_modeleditor_rb PRIVATE ${RUBY_INCLUDE_DIRS})

set_target_properties(openstudio_modeleditor_rb PROPERTIES PREFIX "")
set_target_properties(openstudio_modeleditor_rb PROPERTIES OUTPUT_NAME openstudio_modeleditor)

# TODO: Output directly to Products/openstudio/ruby so it sits next to openstudio.so and others? `${PROJECT_BINARY_DIR}/openstudio/Products/ruby/`
set_target_properties(openstudio_modeleditor_rb PROPERTIES ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_ARCHIVE_OUTPUT_DIRECTORY}/ruby/")
set_target_properties(openstudio_modeleditor_rb PROPERTIES LIBRARY_OUTPUT_DIRECTORY "${CMAKE_LIBRARY_OUTPUT_DIRECTORY}/ruby/")
set_target_properties(openstudio_modeleditor_rb PROPERTIES RUNTIME_OUTPUT_DIRECTORY "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/ruby/")

if(APPLE)
  set_target_properties(openstudio_modeleditor_rb PROPERTIES SUFFIX ".bundle" )
else()
  set_target_properties(openstudio_modeleditor_rb PROPERTIES SUFFIX ".so" )
endif()

if(MSVC)
  # wd4996=no deprecated warnings ; wd5033=register
  set_target_properties(openstudio_modeleditor_rb PROPERTIES COMPILE_FLAGS "/bigobj /wd4996 /wd5033")
endif()

add_custom_command(TARGET openstudio_modeleditor_rb
  POST_BUILD
  COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_SOURCE_DIR}/openstudio_modeleditor.rb $<TARGET_FILE_DIR:openstudio_modeleditor_rb>/openstudio_modeleditor.rb
)

target_link_libraries(openstudio_modeleditor_rb
  ruby_OpenStudioModelEditor
  openstudio_lib
  openstudio_bimserver
  ${QT_WEB_LIBS}
)

if( WIN32 )
  target_link_libraries(openstudio_modeleditor_rb ${RUBY_MINGW_STUB_LIB})
endif()

install(TARGETS openstudio_modeleditor_rb DESTINATION Ruby COMPONENT "RubyAPI")
install(FILES openstudio_modeleditor.rb DESTINATION Ruby COMPONENT "RubyAPI")

if( BUILD_PAT )
  # don't need modeleditor libs in pat
endif()

###############################################################################
#                        T E S T I N G:   C T E S T S                         #
###############################################################################

# find all tests
file(GLOB RUBY_TEST_SRC "test/*.rb")

# TODO: It doesn't not work with this executable that's in build/Ruby-install/bin
# message("RUBY_EXECUTABLE=${RUBY_EXECUTABLE}")

set(_RUBY_POSSIBLE_EXECUTABLE_NAMES
  ruby
  ruby2.5
  ruby2.5.1
  ruby25
  ruby251)

# TODO: this isn't great but I haven't found a better way to locate the system ruby (and avoid the one in build/Ruby-install/)
find_program(SYSTEM_RUBY_EXECUTABLE NAMES ${_RUBY_POSSIBLE_EXECUTABLE_NAMES}
  HINTS
    "/usr"
    "/usr/local"
    "/usr/share/rvm/rubies/ruby-2.5.1/bin/"
    "/usr/share/rvm/rubies/ruby-2.5.2/bin/"
    "/usr/share/rvm/rubies/ruby-2.5.3/bin/"
    "/usr/share/rvm/rubies/ruby-2.5.4/bin/"
    "/usr/share/rvm/rubies/ruby-2.5.5/bin/"
    "$ENV{HOME}/.rvm/rubies/ruby-2.5.1/bin/"
    "$ENV{HOME}/.rvm/rubies/ruby-2.5.2/bin/"
    "$ENV{HOME}/.rvm/rubies/ruby-2.5.3/bin/"
    "$ENV{HOME}/.rvm/rubies/ruby-2.5.4/bin/"
    "$ENV{HOME}/.rvm/rubies/ruby-2.5.5/bin/"
    "C:/Ruby25-x64/bin/"
  NO_DEFAULT_PATH)

if(NOT SYSTEM_RUBY_EXECUTABLE)
  message(WARNING "Your system ruby wasn't found, you won't be able to run the `ctest -R RubyTest` command and the tests won't be created at all.")
else()
  message(STATUS "Found SYSTEM_RUBY_EXECUTABLE=${SYSTEM_RUBY_EXECUTABLE}")

  # add a test for each unit test
  set(RUBY_TEST_REQUIRES "#include test files")
  foreach(f ${RUBY_TEST_SRC})

    file(READ "${f}" CONTENTS)
    string(REGEX MATCHALL "def +test_([A-Za-z_0-9 ]+)" FOUND_TESTS ${CONTENTS})

    foreach(HIT ${FOUND_TESTS})
      string(REGEX REPLACE "def +test_([A-Za-z_0-9]+)" "\\1" TEST_NAME ${HIT})
      string(REGEX MATCH "/?([A-Za-z_0-9 ]+)\\.rb" FILE_NAME ${f})
      string(REGEX REPLACE "/?([A-Za-z_0-9 ]+)\\.rb" "\\1" FILE_NAME ${FILE_NAME})
      if(BUILD_TESTING)

        # Call with Ruby itself
        # TODO: might need -I openstudio_modeleditor_rb instead/as well.
        add_test(NAME "OSApp-RubyTest-${FILE_NAME}-${TEST_NAME}"
          COMMAND "${CMAKE_COMMAND}" -E chdir "${CMAKE_CURRENT_BINARY_DIR}"
          "${SYSTEM_RUBY_EXECUTABLE}" "-I" "$<TARGET_FILE_DIR:openstudio_rb>" "-I" "$<TARGET_FILE_DIR:openstudio_modeleditor_rb>" "${f}" "--name=test_${TEST_NAME}"
        )

        set_tests_properties("OSApp-RubyTest-${FILE_NAME}-${TEST_NAME}" PROPERTIES TIMEOUT 660 )
      endif()
    endforeach()
  endforeach()

endif()
