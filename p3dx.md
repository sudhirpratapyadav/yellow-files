# Installation and setup for P3dx
- Requirement: ROS (noetic)
- Links
	- http://wiki.ros.org/ROSARIA/Tutorials/How%20to%20use%20ROSARIA
	- https://github.com/reedhedges/AriaCoda
- Steps to install required libraries
	- Step 1: Install Aria Coda Library on Ubuntu
		- `git clone https://github.com/reedhedges/AriaCoda.git`
		- `cd AriaCoda`
		- `make -j4`
		- `sudo make install`
	- Step 2: Install ROSARIA package on ROS (noetic)
		- Step 1: Create workspace (here name of workspace is nav_ws)
			- `mkdir -p ~/nav_ws/src`
			- `cd ~/nav_ws/src`
			- `catkin_init_workspace`
			- `cd ~/nav_ws`
			- `catkin_make`
		- Step 2: source setup.bash
			- `echo "source ~/nav_ws/devel/setup.bash/setup.bash" >> ~/.bashrc`
		- Step 3: Build rosaria package
			-  `cd ~/nav_ws/src`
			- `git clone https://github.com/amor-ros-pkg/rosaria.git`
			- `cd ~/nav_ws`
			-  `catkin_make`
- Running
	- `roscore`
	- `rosrun rosaria RosAria`
	- if error comes while accesing usb try giving permissin using below
		- `sudo chmod 777 -R /dev/ttyUSB0`
- Extras
	- Install teleop_twist_keyboard for controlling p3dx using keyboard
		- Link: https://github.com/ros-teleop/teleop_twist_keyboard
		- `rosrun teleop_twist_keyboard teleop_twist_keyboard.py cmd_vel:=my_cmd_vel`
- Overall Commands to run
```
roscore
sudo chmod 777 -R /dev/ttyUSB0
rosrun rosaria rosAria
rosrun teleop_twist_keyboard teleop_twist_keyboard.py cmd_vel:=/RosAria/cmd_vel
```
