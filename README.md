# Example ROS package waypoint_flier

This package was created as an example of how to write basic ROS nodelets. The package is written in C++ and features custom MRS libraries and msgs.

## Functionality

* Desired waypoints are loaded as a matrix from config file
* Service `fly_to_first_waypoint` prepares the UAV by flying to the first waypoint
* Service `start_waypoint_following` causes the UAV to start tracking the waypoints
* Service `stop_waypoint_following` stops adding new waypoints. Flight to the current waypoint is not interrupted.

## Package structure

See [ROS packages](http://wiki.ros.org/Packages)

* `src` directory contains all source files
* `include` directory contains all header files. It is good practice to separate them from source files.
* `launch` directory contains `.launch` files which are used to parametrize the nodelet. Command-line arguments, as well as environment variables, can be loaded from the launch files, the nodelet can be put into the correct namespace (each UAV has its namespace to allow multi-robot applications), config files are loaded, and parameters passed to the nodelet. See [.launch files](http://wiki.ros.org/roslaunch/XML)
* `config` directory contains parameters in `.yaml` files. See [.yaml files](http://wiki.ros.org/rosparam)
* `package.xml` defines properties of the package, such as package name and dependencies. See [package.xml](http://wiki.ros.org/catkin/package.xml)

## Features

* Nodelet initialization
* Subscriber, publisher, and timer initialization
* Service servers and clients initialization
* Loading parameters with `mrs_lib::ParamLoader` class
* Loading matrices with `mrs_lib::ParamLoader` class
* Checking nodelet initialization status in every callback
* Checking whether subscribed messages are coming
* Throttling text output to a terminal
* Thread-safe access to variables using `std::lock_scope()`
* Using `ConstPtr` when subscribing to a topic to avoid copying large messages
* Storing and accessing matrices in `Eigen` classes
* Remapping topics in the launch file

## Coding style

For easy orientation in the code, we have agreed to follow the ROS C++ Style Guide when writing our packages. See [ROSCppStyleGuide](http://wiki.ros.org/CppStyleGuide)

### Naming variables

* Member variables are distinguished from local variables by underscore at the end:
  - `position_x` -  local variable
  - `position_x_` -  member variable
* Also, we distinguish parameters which are loaded as parameters by underscore at the beginning
  - `_simulation_` - parameter
* Descriptive variable names are used. The purpose of the variable should be obvious from the name.
  - `sub_odom_uav_` - member subscriber to uav odometry msg type
  - `pub_goto_` - member subscriber of goto msg type
  - `srv_server_start_waypoints_following_` - member service server for starting following of waypoints
  - `WaypointFlier::callbackTimerCheckSubscribers()` - callback of timer which checks subscribers
  - `mutex_odom_uav_` - mutex locking access to variable containing odometry of the UAV

### Good practices

* Nodelet everything! Nodelets compared to nodes do not need to send whole messages. Multiple nodelets running under the same nodelet manager form one process and messages can be passed as pointers. [Nodelet everything](https://www.clearpathrobotics.com/assets/guides/ros/Nodelet%20Everything.html)
* Do not use raw pointers! Smart pointers from `<memory>` free resources automatically, thus preventing memory leaks.
* Lock access to member variables! Nodelets are multi-thread processes, so it is our responsibility to make our code thread-safe.
  - Use `c++17` `scoped_lock` which unlocks the mutex after leaving the scope. This way, you can't forget to unlock the mutex.
  ```cpp
  {
    std::scoped_lock lock(mutex_odom_uav_);
    odom_uav_ = *msg;
  }
  ```
* Use `ros::Time::waitForValid()` after creating node handle `ros::NodeHandle _nh("~")`
* When a nodelet is initialized, the method `onInit()` is called. In the method, the subscribers are initialized, and callbacks are bound to them. The callbacks can run even before the `onInit()` method ends, which can lead to some variables being still not initialized, parameters not loaded, etc. This can be prevented by using an `is_initialized_`, initializing it to `false` at the beginning of `onInit()` and setting it to true at the end. Every callback should check this variable and continue only when it is `true`.
* Use `mrs_lib::ParamLoader` class to load parameters from launch files and config files. This class checks whether the parameter was actually loaded, which can save a lot of debugging. Furthermore, loading matrices into config files becomes much simpler.
* For printing debug info to terminal use `ROS_INFO()`, `ROS_WARN()`, `ROS_ERROR()` macros. Do not spam the terminal by printing a variable every time a callback is called, use for example `ROS_INFO_THROTTTLE(1.0, "dog")` to print *dog* not more often than every second. Other animals can also be used for debugging purposes.
* If you need to execute a piece of code periodically, do not use sleep in a loop, or anything similar. The ROS API provides `ros::Timer` class for this purposes, which executes a callback every time the timer expires.
* Always check whether all subscribed messages are coming. If not, print a warning. Then you know the problem is not in your nodelet and you know to look for the problem in topic remapping or the node publishing it.
