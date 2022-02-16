cmake_minimum_required(VERSION 3.18) # RUBY_ vars

project(OpenCL-Guide
  LANGUAGES NONE
)

find_package(Ruby QUIET)

if(NOT EXISTS "${RUBY_EXECUTABLE}")
  message(FATAL_ERROR "Ruby not found.")
endif()

if(NOT EXISTS "${AWESOME_BOT_EXECUTABLE}")
  get_filename_component(RUBY_EXECUTABLE_DIR "${RUBY_EXECUTABLE}" DIRECTORY)

  if(WIN32)
    set(AWESOME_BOT_NAMES awesome_bot.cmd awesome_bot.bat awesome_bot.exe)
  else()
    set(AWESOME_BOT_NAMES awesome_bot)
  endif()

  find_program(AWESOME_BOT_EXECUTABLE
    NAMES ${AWESOME_BOT_NAMES}
    HINTS "${RUBY_EXECUTABLE_DIR}"
    PATHS ENV MY_RUBY_HOME
    PATH_SUFFIXES
      bin
      Scripts
  )

  if(NOT EXISTS "${AWESOME_BOT_EXECUTABLE}")
    message(FATAL_ERROR "awesome_bot executable not found. Please install or provide path by specifying AWESOME_BOT_EXECUTABLE")
  endif()
endif()

include(CTest)

file(GLOB_RECURSE MD_FILES
  CONFIGURE_DEPENDS
  chapters/*.md
)

foreach(MD_FILE IN LISTS MD_FILES)
  get_filename_component(MD_FILE_NAME "${MD_FILE}" NAME)
  add_test(
    NAME ${MD_FILE_NAME}
    COMMAND "${AWESOME_BOT_EXECUTABLE}" --allow-dupe "${MD_FILE}" --white-list https://www.youtube.com/,https://github.com/KhronosGroup,https://github.com/microsoft
  )
endforeach()