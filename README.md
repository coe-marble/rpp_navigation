# rpp_navigation

Navigation library for RPP and a ROS 2 package. The navigation design will
closely follow Nav2, with plugin contracts and implementations provided through
RPP.

- `plugin_types/`: Cap'n Proto plugin contracts.
- `plugins/`: C++ and Python plugin implementations.
- `plugins.json`: RPP registration entries for both directories.
- `test/`: unit tests registered with ament and run through colcon.

`package.xml` provides metadata for both ROS 2 and RPP. Register the source
directory with `rpp library register /path/to/rpp_navigation --link`.
The installed library is placed under `share/rpp_navigation` and can also be
registered with RPP.

CI uses the shared RPP ROS 2 Lyrical container on pushes and pull requests to
`main` and `develop`, and supports manual runs. It builds with
`BUILD_TESTING=ON`, runs `colcon test --packages-select rpp_navigation`, and
reports failures with `colcon test-result --verbose`.

The initial package contains no navigation implementations or unit tests.
Add C++ tests with `ament_add_gtest` and Python tests with
`ament_add_pytest_test` in `test/CMakeLists.txt`; CI will pick them up through
colcon.
