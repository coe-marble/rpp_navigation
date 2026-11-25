cmake_minimum_required(VERSION 3.15)
project(rpp_navigation)

find_package(ament_cmake REQUIRED)

install(DIRECTORY plugins plugin_types
  DESTINATION share/${PROJECT_NAME}
  PATTERN ".gitkeep" EXCLUDE
)
install(FILES plugins.json README.md LICENSE
  DESTINATION share/${PROJECT_NAME}
)

if(BUILD_TESTING)
  add_subdirectory(test)
endif()

ament_package()
