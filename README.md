# CarND-Capstone

## Overview
This is the project repo for the final project of the Udacity Self-Driving Car Nanodegree: Programming a Real Self-Driving Car. For more information about the project, see the project introduction [here](https://classroom.udacity.com/nanodegrees/nd013/parts/6047fe34-d93c-4f50-8336-b70ef10cb4b2/modules/e1a23b06-329a-4684-a717-ad476f0d8dff/lessons/462c933d-9f24-42d3-8bdc-a08a5fc866e4/concepts/5ab4b122-83e6-436d-850f-9f4d26627fd9).

For this project, you'll be writing ROS nodes to implement core functionality of the autonomous vehicle system, including traffic light detection, control, and waypoint following! You will test your code using a simulator, and when you are ready, your group can submit the project to be run on Carla.

The following pic shows the structure of the system:
<img src="https://user-images.githubusercontent.com/40875720/55611564-73ce2700-57b8-11e9-853a-5a142fc8c7af.png" width="600">

## Planning
Planning section contains waypoint loader and waypoint updater node. Waypoint loader is provided by Autoware, so in the project we only focus on waypoint updater part.

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

Declare a subscriber to subscribe current_pose topic, then using a call back function to get positionning information.
For more information please see the attached code:

```
# Define a subscriber to subscribe current_pose to get positioning information
# Also declare a pose_cb call back function to do some post processing work after receiving the topic
rospy.Subscriber('/current_pose', PoseStamped, self.pose_cb)

# Define a call back fuction to init pose variable when receiving current_pos topic
def pose_cb(self, msg):
        self.pose = msg
```

### Get basic waypoint information

Declare a subscriber to subscribe base_waypoints topic, then using  a call back function to get base waypoint information.
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

Declare a subscriber to subscribe traffic_waypoint topic, then using a call back function to get the index of stop line. Well prepared for deceleration waypoint topic.
```
rospy.Subscriber('/traffic_waypoint', Int32, self.traffic_cb)

def traffic_cb(self, msg):
        # TODO: Callback for /traffic_waypoint message. Implement
        self.stopline_wp_idx = msg.data
	
```

#### Deceleration waypoints

The purpose of deceleration waypoints function is to decelerate the vehicle until stop if there is a red light in front of the vehicle.
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

The purpose of this section is to find the closest waypoint **in front of** the vehicle. Basically this function can find the index of the closest waypoint, then be well prepared to publish the waypoints to the final_waypoints topic.

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

## Control

Control section contains two parts: Waypoints follower and DBW node. Waypoint follower is used to do near plan planning which means to plan path from one waypoint to the next, the part is provided by Autoware (C++ code, will learn & Modify it latter on). DBW node is our main work in this project. 

<img src="https://user-images.githubusercontent.com/40875720/55665975-a04a7780-587a-11e9-83d0-0ef5901dc4a2.PNG" width="400">

Once messages are being published to /final_waypoints, the vehicle's waypoint follower will publish twist commands to the /twist_cmd topic. The goal for this part of the project is to implement the drive-by-wire node (dbw_node.py) which will subscribe to /twist_cmd and use various controllers to provide appropriate throttle, brake, and steering commands. These commands can then be published to the following topics:

* /vehicle/throttle_cmd
* /vehicle/brake_cmd
* /vehicle/steering_cmd

DBW node part contains the following items:
* Get DBW enable information
* Get vehilce speed information
* Get linear & angular speed information
* Control throttle, brake and steering

### Get DBW enable information

Since a safety driver may take control of the car during testing, you should not assume that the car is always following your commands. If a safety driver does take over, your PID controller will mistakenly accumulate error, so you will need to be mindful of DBW status. The DBW status can be found by subscribing to /vehicle/dbw_enabled.
When operating the simulator please check DBW status and ensure that it is in the desired state. DBW can be toggled by clicking "Manual" in the simulator GUI.
This part is quite simple, we only need to get signal from the message. For more information please refer the code as below:

```
rospy.Subscriber('/vehicle/dbw_enabled', Bool, self.dbw_enabled_cb)
def dbw_enabled_cb(self, msg):
        self.dbw_enabled = msg
```

### Get vehicle speed information

Current speed is important for PID cotroller to calculate the speed error. This part is also quite simple, please find more information in the below code:
```
rospy.Subscriber('/current_velocity', TwistStamped, self.velocity_cb)
def velocity_cb(self, msg):
        self.current_vel = msg.twist.linear.x 
```
### Get linear & angular speed information

The part is to get the desire linear and angular vehilce from waypoint_follower. Please find more information in the below code:
```
rospy.Subscriber('/twist_cmd', TwistStamped, self.twist_cb)
def twist_cb(self, msg):
        self.linear_vel = msg.twist.linear.x
        self.angular_vel = msg.twist.angular.z
```

### Control throttle, brake and steering

#### Trottle control

In the project, we are trying to control a traditional vehicle, so we need to control throttle, brake and steering, but on the other hand, if we control new engery vehilce, then we only need to control speed and steering, it will be easier. Back to our topic, we need to following the following steps to control throttle:

* Filter current speed using low pass filter
* Calculate velocity error (desired linear velocity minus current linear velocity)
* Optimize velocity error (Consider deceleration limit & acceleration speed)
* Get sample time (sample time is used to calculate derivative)
* Call PID function to get the throttle value
```
        current_linear_velocity = self.vel_lpf.filt(current_linear_velocity)
        
        velocity_error = desired_linear_velocity - current_linear_velocity
	velocity_error = max(self.decel_limit,velocity_error)
	velocity_error = min(velocity_error,self.accel_limit)

        self.last_vel = current_linear_velocity
        
        # find the time duration and a new timestamp
        current_time = rospy.get_time()
        sample_time = current_time - self.last_time
        self.last_time = current_time
        
        throttle = self.throttle_controller.step(velocity_error, sample_time)
```

#### Brake control

Since we are using traditional vehicle, we need to apply some brake touque to stop the vehicle.

```
        brake = 0       
        if desired_linear_velocity <= 0.3:
            throttle = 0
            brake = 0.4*self.max_brake_torque
	    if current_linear_velocity <= 0.3:
		brake = self.max_brake_torque # Torque N*m
	brake = min(brake,self.max_brake_torque)
	brake = max(0.0, brake)

	brake = self.brake_lpf.filt(brake)
```

#### Steering control

The purpose of this part is to calculate the steering angle of the wheel. Basically it contains the following steps:

* Get the current linear speed (low pass filter)
* Calculate turn radius (v = wr then r = v/w)
* Call get angle function to get steering angle
```
class YawController(object):
    def __init__(self, wheel_base, steer_ratio, min_speed, max_lat_accel, max_steer_angle):
        self.wheel_base = wheel_base
        self.steer_ratio = steer_ratio
        self.min_speed = min_speed
        self.max_lat_accel = max_lat_accel

        self.min_angle = -max_steer_angle
        self.max_angle = max_steer_angle

    def get_angle(self, radius):
        angle = atan(self.wheel_base / radius) * self.steer_ratio
        return max(self.min_angle, min(self.max_angle, angle))

    def get_steering(self, linear_velocity, angular_velocity, current_velocity):
        angular_velocity = current_velocity * angular_velocity / linear_velocity if abs(linear_velocity) > 0. else 0.

        if abs(current_velocity) > 0.1:
            max_yaw_rate = abs(self.max_lat_accel / current_velocity);
            angular_velocity = max(-max_yaw_rate, min(max_yaw_rate, angular_velocity))

        return self.get_angle(max(current_velocity, self.min_speed) / angular_velocity) if abs(angular_velocity) > 0. else 0.0;

self.yaw_controller = YawController(
            self.wheel_base, self.steer_ratio, 0.1,
            self.max_lat_accel, self.max_steer_angle) 

current_linear_velocity = self.vel_lpf.filt(current_linear_velocity)

steering = self.yaw_controller.get_steering(desired_linear_velocity,desired_angular_velocity, current_linear_velocity)
	
```

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



