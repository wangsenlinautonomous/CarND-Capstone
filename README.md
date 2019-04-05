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

Declare a subscriber to subscrib current_pose topic, then using a call back function to get positionging information.
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

Declare a subscriber to subscrib base_waypoints topic, then using  a call back function to get base waypoint information.
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

### Get traffic light information

Declare a subscriber to subscribe traffic_waypoint topic, then using a call back function to get the index of stopline. Well prepared for deceleration waypoint topic.
```
rospy.Subscriber('/traffic_waypoint', Int32, self.traffic_cb)

def traffic_cb(self, msg):
        # TODO: Callback for /traffic_waypoint message. Implement
        self.stopline_wp_idx = msg.data
	
```

#### Deceleration waypoints

The purpose of deceleration waypoints function is to decelerate the vehicle until stop if there is a red light infornt of the vehicle.
For this purpose we only need to update the vehicle speed(twist.linear) of the each waypoints.
In the case, we will create a new waypoint list to update the vehicle speed, otherwise it will over write the original vehicle speed, then the behavior of the vehicle can be strange in the following loops.

Also to get a smooth vehicle speed is quite important. we use "vel = math.sqrt(2 * MAX_DECEL * SAFETY_FACTOR * dist)" to calculate the velocity according to dist. The result is showing in the following pic:

<img src="https://user-images.githubusercontent.com/40875720/55624827-a4728880-57d9-11e9-911e-08ea61ea5bd4.PNG" width="400">

```
def decelerate_waypoints(self, waypoints, closest_idx):
        result = []
        for i, wp in enumerate(waypoints):
            new_point = Waypoint()
            new_point.pose = wp.pose
	    
	    # Should stop in front of the stop line, otherwise it will be dangerous to break law
            stop_idx = max(self.stopline_wp_idx - closest_idx - 2, 0)  
	    
            # the car stops at the line
            dist = self.distance(waypoints, i, stop_idx)
            vel = math.sqrt(2 * MAX_DECEL * SAFETY_FACTOR * dist)
            if vel < 1.0:
                vel = 0.0

            new_point.twist.twist.linear.x = min(vel, wp.twist.twist.linear.x)
            result.append(new_point)

        return result
```


### Publish final waypoints

#### KD Tree introduction

In computer science, a k-d tree (short for k-dimensional tree) is a space-partitioning data structure for organizing points in a k-dimensional space. k-d trees are a useful data structure for several applications, such as searches involving a multidimensional search key (e.g. range searches and nearest neighbor searches). k-d trees are a special case of binary space partitioning trees.

Basically, it contains two parts: Build and Search. In our case, I use "self.waypoint_tree = KDTree(self.waypoints_2d)" function to build KD tree, use "waypoint_tree.query([x, y], 1)[1]" function to search.

Find more information in the following links:

https://baike.baidu.com/item/kd-tree/2302515?fr=aladdin

https://en.wikipedia.org/wiki/K-d_tree

#### Get the closest waypoints

The purpose of this section is to find the closest waypoint **in front of** the vehicle. Basically this funciton can find the index of the closest waypoint, then be well prepared to publish the waypoints to the final_waypoints topic.

For more details please refer to the code below,I made some comments to the code
```
def get_closest_waypoint_idx(self):
        # Get the coordinates of our car
        x = self.pose.pose.position.x
        y = self.pose.pose.position.y
	
        # Get the index of the closest point
        closest_idx = self.waypoint_tree.query([x, y], 1)[1]

        # Check if closest is ahead or behind vehicle
        closest_coord = self.waypoints_2d[closest_idx]
        prev_coord = self.waypoints_2d[closest_idx - 1]

        # Equation for hyperplane through closest_coords
        cl_vect = np.array(closest_coord)
        prev_vect = np.array(prev_coord)
        pos_vect = np.array([x, y])

        val = np.dot(cl_vect - prev_vect, pos_vect - cl_vect)
        if val > 0: # Our car is in front of the closest waypoint
            closest_idx = (closest_idx + 1) % len(self.waypoints_2d)

        return closest_idx
```

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



