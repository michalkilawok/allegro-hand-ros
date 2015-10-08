allegro_hand_ros beta2, unofficial fork
=======================================

This is an unofficial fork of SimLab's allegro [hand ros package][1].

[1]: https://github.com/simlabrobotics/allegro_hand_ros

It improves upon the SimLab version by providing a catkin-ized version,
simplifies the launch file structure,
updates the package/node names to have a more consistent structure, 
improves the build process by creating a common driver, introduces an
AllegroNode C++ class that reduces the amount of duplicated code.

It also provides the BHand library directly in this package (including both
32-bit and 64-bit versions, though 32-bit systems will need to update the
symlink manually).

At this point no effort has been made to be backwards compatible. Some of the
non-compatible changes between the two version are:

 - Put all of the controllers into one *package* (allegro_hand_controllers) and
   made each controller a different node (allegro_node_XXXX): grasp, pd, velsat.
 - Single launch file with arguments instead of multiple launch files with
   repeated code.
 - Both the parameter and description files are now ROS packages, so that
   `rospack find` works with them.
 - These packages will likely not work with pre-hydro versions (only tested on
   indigo so far, please let me know if this works on other distributions).

Launch file instructions:
------------------------

There is now a single file,
[allegro_hand.launch](allegro_hand_controllers/launch/allegro_hand.launch)
that starts the hand. It takes many arguments, but at a minimum you must specify
the handedness:

    roslaunch allegro_hand_controllers allegro_hand.launch HAND:=right

Optional (recommended) arguments:

          NUM:=0|1|...
          ZEROS:=/path/to/zeros_file.yaml
          CONTROLLER:=grasp|pd|velsat
          RESPAWN:=true|false   Respawn controller if it dies.
          KEYBOARD:=true|false  (default is true)
          AUTO_CAN:=true|false  (default is true)
          CAN_DEVICE:=/dev/pcanusb1 | /dev/pcanusbNNN  (ls -l /dev/pcan* to see open CAN devices)
          VISUALIZE:=true|false  (Launch rviz)
          
Note on `AUTO_CAN`: There is a nice script `detect_pcan.py` which automatically
finds an open `/dev/pcanusb` file. If instead you specify the can device
manually (`CAN_DEVICE:=/dev/pcanusbN`), make sure you *also* specify
`AUTO_CAN:=false`. Obviously, automatic detection cannot work with two hands.

The second launch file is for visualization, it is included in
`allegro_hand.launch` if `VISUALIZE:=true`. Otherwise, it can be useful to run
it separately (with `VISUALIZE:=false`), for example if you want to start rviz separately
(and keep it running):

    roslaunch allegro_hand_controllers allegro_viz.launch HAND:=right
    
Note that you should also specify the hand `NUM` parameter in the viz launch if
the hand number is not zero.

Packages
--------

 * **allegro_hand** Metapackage.
 * **allegro_hand_common** Driver for talking with the allegro hand.
 * **allegro_hand_controllers** Different nodes that actually control the hand. 
 The AllegroNode class handles all the generic driver comms, each class then
 implements `computeDesiredTorque` differently (and can have various topic 
 subscribers):
   * grasp: Apply various pre-defined grasps, including gravity compensation.
   * pd: Joint space control: save and hold positions.
   * velsat: velocity saturation joint space control (supposedly experimental)
 * **allegro_hand_description** xacro descriptions for the kinematics of the
     hand, rviz configuration and meshes.
 * **allegro_hand_keyboard** Node that sends the commanded grasps. All commands 
     are available with the grasp controller, only some are available with the 
     other controllers. 
 * **allegro_hand_parameters** All necessary parameters for loading the hand:
   * gains_pd.yaml: Controller gains for PD controller.
   * gains_velSat.yaml: Controller gains and parameters for velocity saturation 
           controller.
   * initial_position.yaml: Home position for the hand.
   * zero.yaml: Offset and servo directions for each of the 16 joints, and some 
           meta information about the hand.
   * zero_files/ Zero files for all hands.
 * **bhand** Library files for the predefined grasps, available in 32 and 64 bit
     versions. 64 bit by default, update symlink for 32 bit.

Note on polling (from SimLabs): The preferred sampling method is utilizing the
Hand's own real time clock running @ 333Hz by polling the CAN communication
(polling = true, default). In fact, ROS's interrupt/sleep combination might
cause instability in CAN communication resulting unstable hand motions. 


Useful Links
------------

 * [Allegro Hand wiki](http://www.simlab.co.kr/AllegroHand/wiki).
 * [ROS wiki for original package](http://www.ros.org/wiki/allegro_hand_ros).
 

Controlling More Than One Hand
------------------------------

When running more than one hand using ROS, you must specify the number of the
hand when launching.

    roslaunch allegro_hand.launch HAND:=right ZEROS:=parameters/zero0.yaml NUM:=0 CAN_DEVICE:=/dev/pcan0 AUTO_CAN:=false
    
    roslaunch allegro_hand.launch HAND:=left  ZEROS:=parameters/zero1.yaml NUM:=1 CAN_DEVICE:=/dev/pcan1 AUTO_CAN:=false


Known Issues:
-------------

While all parameters defining the hand's motor/encoder directions and offsets
fall under the enumerated "allegroHand_#" namespaces, the parameter
"robot_description" defining the kinematic structure and joint limits remains
global. When launching a second hand, this parameter is overwritten. I have yet
to find a way to have a separate enumerated "robot_decription" parameter for
each hand. If you have any info on this, please advise.
