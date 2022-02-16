cmake_minimum_required(VERSION 3.18) # RUBY_ vars

project(OpenCL-Guide
  LANGUAGES NONE
)

if(NOT EXISTS "${Awesome_bot_EXECUTABLE}")
  find_package(Ruby QUIET)

  if(NOT EXISTS "${RUBY_EXECUTABLE}")
    message(FATAL_ERROR "Ruby not found.")
  endif()

  get_filename_component(RUBY_EXECUTABLE_DIR "${RUBY_EXECUTABLE}" DIRECTORY)

  if(WIN32)
    set(GEM_NAMES gem.cmd gem.bat gem.exe)
  else()
    set(GEM_NAMES gem)
  endif()

  find_program(GEM_EXECUTABLE
    NAMES ${GEM_NAMES}
    HINTS "${RUBY_EXECUTABLE_DIR}"
    PATHS ENV MY_RUBY_HOME
    PATH_SUFFIXES
      bin
      Scripts
    NO_CMAKE_PATH
    NO_CMAKE_ENVIRONMENT_PATH
    NO_SYSTEM_ENVIRONMENT_PATH
    NO_CMAKE_SYSTEM_PATH
  )

  if(NOT EXISTS "${GEM_EXECUTABLE}")
    message(FATAL_ERROR "gem not found.")
  endif()

  execute_process(
    COMMAND "${GEM_EXECUTABLE}" install awesome_bot
    RESULT_VARIABLE AWESOME_BOT_RESULT_VAR
    OUTPUT_VARIABLE AWESOME_BOT_OUTPUT_VAR
    ERROR_VARIABLE  AWESOME_BOT_ERROR_VAR
    OUTPUT_STRIP_TRAILING_WHITESPACE
  )

  if(AWESOME_BOT_RESULT_VAR)
    message(SEND_ERROR "Failed to install awesome_bot")
    message(SEND_ERROR "AWESOME_BOT_RESULT_VAR: ${AWESOME_BOT_RESULT_VAR}")
    message(SEND_ERROR "${AWESOME_BOT_OUTPUT_VAR}")
    message(FATAL_ERROR "${AWESOME_BOT_ERROR_VAR}")
  endif()

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
    NO_CMAKE_PATH
    NO_CMAKE_ENVIRONMENT_PATH
    NO_SYSTEM_ENVIRONMENT_PATH
    NO_CMAKE_SYSTEM_PATH
  )
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