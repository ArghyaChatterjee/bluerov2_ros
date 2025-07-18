# BlueROV2 ROS

Orca is a [ROS](http://ros.org) driver for [BlueRobotics BlueROV2](https://www.bluerobotics.com/store/rov/bluerov2/).
Version 0.1.0  supports basic ROV functions.

Orca requires hardware modifications to the BlueROV2.
It does not use the Pixhawk controller that comes with the BlueROV2, and does not support [mavros](http://wiki.ros.org/mavros), [ArduSub](https://www.ardusub.com/) or [QGroundControl](http://qgroundcontrol.com/).

> **NOTE:** development has shifted to [Orca3](https://github.com/clydemcqueen/orca3),
> which uses [ROS2 Navigation](https://navigation.ros.org/index.html) for mission planning and execution.

## Tested hardware

* [BlueRobotics BlueROV2](https://www.bluerobotics.com/store/rov/bluerov2/), with the included Raspberry Pi 3, Fathom-X and Bar30
* [Pololu Maestro 18-Channel USB Servo Controller](https://www.pololu.com/product/1354)
* [PhidgetSpatial Precision 3/3/3 High Resolution IMU](https://www.phidgets.com/?tier=3&catid=10&pcid=8&prodid=32)
* Xbox One gamepad

## Tested Software

* Tested with Ubuntu 18.04 and ROS Melodic

## Simulation Demonstration
Here is a short demonstration. For better quality, see the demo on [youtube](https://www.youtube.com/watch?v=whqWEsKmHw8). 
<p align="center">
    <img src="Blue_ROV2.gif", width="800">
</p>

## Simulation

Orca runs in [Gazebo](http://gazebosim.org/), a SITL (software-in-the-loop) simulator.
Use the instructions below to install ROS, Gazebo and Orca on your desktop or laptop.

Install [ROS Melodic](http://wiki.ros.org/Installation/Ubuntu).
Select `ros-melodic-desktop-full`; this will install Gazebo 9 as well.

Install these additional packages:
~~~~
sudo apt-get update
sudo apt install ros-melodic-imu-tools ros-melodic-robot-localization
sudo apt-get install -y ros-melodic-gscam
sudo apt-get install -y gstreamer1.0-tools libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev libgstreamer-plugins-good1.0-dev
sudo apt-get install -y libqt5gstreamer-dev
~~~~

Create a catkin workspace and install orca:
~~~~
source /opt/ros/melodic/setup.bash
mkdir -p ~/orca_catkin_ws/src
cd ~/orca_catkin_ws/src
git clone https://github.com/ArghyaChatterjee/orca.git
cd ..
catkin_make
source devel/setup.bash
~~~~

Attach the joystick and run the simulation:
~~~~
roslaunch orca_gazebo gazebo.launch --screen
~~~~
### Start Simulation
Click the `play` button inside Gazebo simulator to start the simulation. You will be able to see the ROV started to publish the transforms and robot model inside Rviz.

### Manual Control
For manual control, plug in your gamepad, hit the [menu button](https://support.xbox.com/en-US/xbox-one/accessories/xbox-one-wireless-controller) (in my case, it's Right Back, RB) to arm the thrusters and start driving around. It's important to map the keys if you are using different joystick. Here's how the buttons are mapped:
* Left stick up/down is forward/reverse
* Left stick left/right is yaw left/right
* Right stick up/down is ascend/descend
* Right stick left/right is strafe left/right
* Menu button: arm
* View button: disarm
* A button: manual
* X button: hold heading
* B button: hold depth
* Y button: hold heading and depth

### Automatic Control
For Automatic Control, Try the 2D Nav Goal button from builtin Rviz panel. You can also publish goal to the `/move_base_simple/goal` topic. Open a new terminal and publish `geometry_msgs/PoseStamped` message like this:
```
rostopic pub /move_base_simple/goal geometry_msgs/PoseStamped "{header: {frame_id: 'odom'}, pose: {position: {x: 0.5, y: 0.5, z: 2.5}, orientation: {x: 0.5, y: 0.5, z: 0.5, w: 1.0}}}"
```
## Design

3 ways has been considered to ROSify a BlueROV2:

* Use the existing hardware (Pixhawk) and software (ArduSub), and use [mavros](http://wiki.ros.org/mavros) to move messages from the MAV message bus to/from ROS.
See [BlueRov-ROS-playground](https://github.com/patrickelectric/bluerov_ros_playground) for a good example of this method.
This is the fastest way to integrate ROS with the BlueROV2.
* Port ROS to the Pixhawk and NuttX, and run a ROS-native driver on the Pixhawk. I don't know of anybody working on this.
* Provide a ROS-native driver running on Linux, such as the Raspberry Pi 3.
You'll need to provide a small device controller, such as the Pololu Maestro, and an IMU, such as the Phidgets IMU.
This is the Orca design.

There are 7 projects:
* `orca_msgs` provides message types
* `orca_description` provides robot description files
* `orca_driver` provides the interface between the hardware and ROS (not required for simulations)
* `orca_base` provides the ROV functionality
* `orca_topside` provides the topside environment (rviz is used instead of QGroundControl)
* `orca_gazebo` provides the simulation environment
* `orca_vision` provides a stereo vision system (experimental)

## Hardware modifications

This is rough sketch of the hardware modifications I made to the BlueROV2. YMMV.

* Remove the Pixhawk and Pixhawk power supply
* Install the Maestro and connect to Pi3 via USB; set the jumper to isolate the Maestro power rail from USB-provide power
* Install the IMU and connect to the Pi3 via USB
* Connect the Bar30 to the Pi3 I2C and 3.3V power pins
* Connect the ESCs to the Maestro; cut the power wire on all but one ESC
* Connect the camera tilt servo to the Maestro
* Connect the lights signal wire to the Maestro; provide power and ground directly from the battery
* Connect the leak detector to a Maestro digital input
* Build a voltage divider to provide a voltage signal from the battery (0-17V) to a Maestro analog input (0-5V)

Orca power budget:

* The BlueROV2 5V 3A regulated power supply provides power for the RPi (2.5A), the Maestro (50mA), the leak detector (20mA), the Bar30 (2mA), the IMU (55mA) and the RPi camera (250mA)
* The Fathom-X, LED lights and ESCs are powered directly from the battery
* The camera tilt servo is powered from the 5V 500mA regulator on one (just one!) of the ESCs. Set the jumpers on the Maestro to isolate the servo rail from the USB-provided power

Notes on the RPi3 power needs:

* The RPi3 needs a 5V 2.5A power supply. It can provide 1.2A to all USB devices
* The RPi3 can provide 50mA across all GPIO pins, but just 16mA for any particular pin
* Note that the RPi3 pins are 3.3V, not 5V
