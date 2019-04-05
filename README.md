# CarND-Capstone

## Overview
This is the project repo for the final project of the Udacity Self-Driving Car Nanodegree: Programming a Real Self-Driving Car. For more information about the project, see the project introduction [here](https://classroom.udacity.com/nanodegrees/nd013/parts/6047fe34-d93c-4f50-8336-b70ef10cb4b2/modules/e1a23b06-329a-4684-a717-ad476f0d8dff/lessons/462c933d-9f24-42d3-8bdc-a08a5fc866e4/concepts/5ab4b122-83e6-436d-850f-9f4d26627fd9).

For this project, you'll be writing ROS nodes to implement core functionality of the autonomous vehicle system, including traffic light detection, control, and waypoint following! You will test your code using a simulator, and when you are ready, your group can submit the project to be run on Carla.

The following pic shows the structure of the system:
<img src="https://user-images.githubusercontent.com/40875720/55611564-73ce2700-57b8-11e9-853a-5a142fc8c7af.png" width="600">

## Planning
Planning section contains waypoint loader and waypoint updater node. Waypoint lader is provided by Autoware, so in the project we only foucs on waypoint updater part.

The basic idea of waypoint updater is to get position information from current_pose topic and get basic waypoints information from base_waypoint topic then publish final waypoint to final_waypoint topic accordingly. Also waypoint updater can consider traffic light and obstacle situations

So waypoint updater contains the following items:
* Get position information
* Get basic waypoint information
* Get traffic light information
  - Decelerate waypoints
* Publish final waypoint
  - KD tree introduction
  - Get the closest waypoint



<img src="https://user-images.githubusercontent.com/40875720/55613528-73845a80-57bd-11e9-8cf4-641de58c8f7f.PNG" width="400">

Now I'd like to go deeper to some subcomponents of waypoint updater

### Get position information

I declare a subscriber to subscrib current_pose topic, then using a call back function to get positionging information.
For more information please see the attached code:

```
# Define a subscriber to subscrib current_pose to get positioning information
# Also declare a pose_cb call back function to do some post processing work after receiving the topic
rospy.Subscriber('/current_pose', PoseStamped, self.pose_cb)

# Define a call back fuction to init pose variable when receiving current_pos topic
def pose_cb(self, msg):
        self.pose = msg
```

### Get basic waypoint information

I declare a subscriber to subscrib base_waypoints topic, then using  a call back function to get base waypoint information.
Then transfer waypoints from 3D to 2D
```
from scipy.spatial import KDTree

rospy.Subscriber('/base_waypoints', Lane, self.waypoints_cb)

def waypoints_cb(self, waypoints):
        #Get the basic waypoints from waypoint loader.This action only need to be done once
        self.base_waypoints = waypoints
	
        #Got 2d waypoints from 3d waypoints
        if not self.waypoints_2d:
	     # Only take x and y in to consideration to get 2D waypoints
             self.waypoints_2d = [[waypoint.pose.pose.position.x, waypoint.pose.pose.position.y] for waypoint in waypoints.waypoints]
             # Build KD tree by using KDTree function, KDTree function scipy.spatial lib
	     self.waypoint_tree = KDTree(self.waypoints_2d)
```

### Publish final waypoint

The result is showing as below:

<img src="https://user-images.githubusercontent.com/40875720/55612987-22279b80-57bc-11e9-83e4-91e97e06c8ec.PNG" width="400">

## Environment Setup
Please use **one** of the two installation options, either native **or** docker installation.

### Native Installation

* Be sure that your workstation is running Ubuntu 16.04 Xenial Xerus or Ubuntu 14.04 Trusty Tahir. [Ubuntu downloads can be found here](https://www.ubuntu.com/download/desktop).
* If using a Virtual Machine to install Ubuntu, use the following configuration as minimum:
  * 2 CPU
  * 2 GB system memory
  * 25 GB of free hard drive space

  The Udacity provided virtual machine has ROS and Dataspeed DBW already installed, so you can skip the next two steps if you are using this.

* Follow these instructions to install ROS
  * [ROS Kinetic](http://wiki.ros.org/kinetic/Installation/Ubuntu) if you have Ubuntu 16.04.
  * [ROS Indigo](http://wiki.ros.org/indigo/Installation/Ubuntu) if you have Ubuntu 14.04.
* [Dataspeed DBW](https://bitbucket.org/DataspeedInc/dbw_mkz_ros)
  * Use this option to install the SDK on a workstation that already has ROS installed: [One Line SDK Install (binary)](https://bitbucket.org/DataspeedInc/dbw_mkz_ros/src/81e63fcc335d7b64139d7482017d6a97b405e250/ROS_SETUP.md?fileviewer=file-view-default)
* Download the [Udacity Simulator](https://github.com/udacity/CarND-Capstone/releases).

### Docker Installation
[Install Docker](https://docs.docker.com/engine/installation/)

Build the docker container
```bash
docker build . -t capstone
```

Run the docker file
```bash
docker run -p 4567:4567 -v $PWD:/capstone -v /tmp/log:/root/.ros/ --rm -it capstone
```

### Port Forwarding
To set up port forwarding, please refer to the [instructions from term 2](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/0949fca6-b379-42af-a919-ee50aa304e6a/lessons/f758c44c-5e40-4e01-93b5-1a82aa4e044f/concepts/16cf4a78-4fc7-49e1-8621-3450ca938b77)

### Usage

1. Clone the project repository
```bash
git clone https://github.com/udacity/CarND-Capstone.git
```

2. Install python dependencies
```bash
cd CarND-Capstone
pip install -r requirements.txt
```
3. Make and run styx
```bash
cd ros
catkin_make
source devel/setup.sh
roslaunch launch/styx.launch
```
4. Run the simulator

### Real world testing
1. Download [training bag](https://s3-us-west-1.amazonaws.com/udacity-selfdrivingcar/traffic_light_bag_file.zip) that was recorded on the Udacity self-driving car.
2. Unzip the file
```bash
unzip traffic_light_bag_file.zip
```
3. Play the bag file
```bash
rosbag play -l traffic_light_bag_file/traffic_light_training.bag
```
4. Launch your project in site mode
```bash
cd CarND-Capstone/ros
roslaunch launch/site.launch
```
5. Confirm that traffic light detection works on real life images



