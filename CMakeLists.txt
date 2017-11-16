cmake_minimum_required(VERSION 2.8)

include(project-config.cmake)

include(${CMAKE_TOOLCHAIN_FILE})

project(${PROJECT_NAME})

# Robot target
# For quick list run
# find src -iname "*.cpp"
# in project root
#set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++14")
set(SOURCE_FILES src/Robot.cpp)
add_executable(${PROJECT_NAME} ${SOURCE_FILES})
find_library(WPILIB wpi HINT ${WPIBASE}/lib)
target_link_libraries(${PROJECT_NAME} ${WPILIB})
include_directories(${WPIBASE}/include)

# Test target
include(ExternalProject)
ExternalProject_Add(check
  PREFIX ${CMAKE_CURRENT_SOURCE_DIR}/test
  SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/test
  BINARY_DIR ${CMAKE_CURRENT_BINARY_DIR}/test
  STAMP_DIR ${CMAKE_CURRENT_BINARY_DIR}/test/stamp
  INSTALL_COMMAND ""
  TEST_AFTER_INSTALL 1
  TEST_COMMAND ${CMAKE_CURRENT_BINARY_DIR}/test/check)
set_target_properties(check PROPERTIES EXCLUDE_FROM_ALL TRUE)

# Deploy target
find_file(ROBOTCOMMAND ant/robotCommand ${WPIBASE})
set(TARGET_USER lvuser)
set(TARGET_DIR /home/lvuser)
set(TARGET "roborio-${TEAM_NUMBER}-frc.local")

find_program(SCP_EXECUTABLE ${SCP_EXECUTABLE_NAME})
find_program(SSH_EXECUTABLE ${SSH_EXECUTABLE_NAME})

if (SCP_EXECUTABLE AND SSH_EXECUTABLE)
add_custom_target(deploy
  COMMAND ${CMAKE_COMMAND} -E echo "Removing old program..."
  COMMAND ${SSH_EXECUTABLE} "${TARGET_USER}@${TARGET}" "rm -f $TARGET_DIR/FRCUserProgram"
  COMMAND ${CMAKE_COMMAND} -E echo "Copying new program over..."
  COMMAND ${SCP_EXECUTABLE} "${PROJECT_NAME}" "${TARGET_USER}@${TARGET}:${TARGET_DIR}/FRCUserProgram"
  COMMAND ${CMAKE_COMMAND} -E echo "Stopping netconsole-host"
  COMMAND ${SSH_EXECUTABLE} "${TARGET_USER}@${TARGET}" "sh -c 'killall -q netconsole-host || :'"
  COMMAND ${CMAKE_COMMAND} -E echo "Copying over robotCommand..."
  COMMAND ${SCP_EXECUTABLE} "${ROBOTCOMMAND}" "${TARGET_USER}@${TARGET}:${TARGET_DIR}"
  COMMAND ${CMAKE_COMMAND} -E echo "Cleaning up..."
  COMMAND ${SSH_EXECUTABLE} "${TARGET_USER}@${TARGET}" "sh -c '. /etc/profile.d/natinst-path.sh; chmod a+x $TARGET_DIR/FRCUserProgram; /usr/local/frc/bin/frcKillRobot.sh -t -r; sync'" DEPENDS ${PROJECT_NAME})
set_target_properties(deploy PROPERTIES EXCLUDE_FROM_ALL TRUE)

endif()

# Doxygen target
find_package(Doxygen)
find_file(DOXYFILE_IN Doxyfile.in  ${CMAKE_CURRENT_SOURCE_DIR})
if(DOXYGEN_EXECUTABLE AND DOXYFILE_IN)

configure_file(Doxyfile.in
  ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile @ONLY)
add_custom_target(doxygen
  ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)
set_target_properties(doxygen PROPERTIES EXCLUDE_FROM_ALL TRUE)

endif()
