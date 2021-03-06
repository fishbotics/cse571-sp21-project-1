In this project you will be implementing robot localization methods such as Extended Kalman Filter (EKF) and Particle Filter (PF). Given a map, your localization method will take sensor measurements of landmarks observed by the robot, and control updates from teleoperation of the robot, to localize the robot in the map. 

For this project you will be working in a group of 3. Please use the Google Sheet and Discord channel to form a group of 3. See the post about this on [Edstem](https://edstem.org/us/courses/4944/discussion/369568).

Once you form a team please drop an email to us (afishman@cs.washington.edu, rohjunha@cs.washington.edu, kdesingh@cs.washington.edu) along with a `team-name`. 

# Deadline 05/07/2021, 11:59 PM Pacific. 
- Project report submission on gradescope.
- Code submission will be via github repositories. Each group should create a private repo (see Task-1 step-1). Name their repos to `cse571-sp21-project-1-{team-name}` give the TAs (github ids - fishbotics, rohjunha, desinghkar) access to these github repositories. 

# Recommended exercises:

- Understand the folder structure and how to create new packages, build and run the packages on local as well as on robot, please see the [documentation](https://docs.duckietown.org/daffy/duckietown-robotics-development/out/dt_infrastructure.html).

- Understand ROS topics, subscriber, publisher, tf message, broadcaster, Markers for drawing in RViz. Please see the [documentation](http://wiki.ros.org/ROS/Tutorials)

- ROS provides packages that can be used draw shapes in RViz. You can use this to trace different paths taken by the robot. Please see the documentation [link1](https://stackoverflow.com/questions/63774548/plot-multiple-paths-in-rviz) and [link2](https://www.programcreek.com/python/example/88812/visualization_msgs.msg.Marker)


# Overview of cse571-sp21-project-1 repository

### cse571-sp21-project-1 repo contains four packages that you won't have to modify:

- `my_at_localization` that localizes the AprilTAGs seen by the robot
- `my_static_tf` that publishes coordinate transforms (position and orientation) of all the landmarks in predetermined maps. These coordinate frames are in the form of ROS `\tf` messages
- `my_viz` that can publish map information and transformations in a visualizer (ROS visualizer called RViz).
- `image_processing` that contains utility functions to process images.

### cse571-sp21-project-1 repo contains packages that you will have to edit in order to complete the project:

- `sensor_fusion_localization` that localizes the robot based on the velocity control and AprilTAG localization.

You can see them in the `./packages/` path.
The source code for each of these packages are inside their respective `src/` folders. For example for `my_at_localization` you will see the source code in the path `./packages/my_at_localization/src/`.


# Outline of tasks in this project:

- TASK-1 Build the provided packages, run it on the robot, record what you can visualize.
- TASK-2 Create a physical `map-1` using duckietown supplies.
- TASK-3 Implement `sensor_fusion_localization` package and test on `map-1`.
- TASK-4 Create a physical `map-2` using duckietown supplies.
- TASK-5 Test `sensor_fusion_localization` package on `map-2`.
- TASK-6 Write up a report.


# Detailed description of tasks:

## TASK-1 Build the provided packages, run it on the robot, record what you can visualize.

After completing project 0.5. You must be able to communicate with your robot from your local computer using `dts shell`

### 1.  Create a private empty repo on github, clone this public repo, change the remote server and push to your private repo.

```
git clone git@github.com:fishbotics/cse571-sp21-project-1.git cse571-sp21-project-{TEAM_NAME}
cd cse571-sp21-project-{TEAM_NAME}/
git remote set-url origin <path to empty private repo>
git remote add upstream git@github.com:fishbotics/cse571-sp21-project-1.git
git push origin
```

### 2. Build the above mentioned packages on the duckiebot

`dts devel build -f -H {ROBOT_NAME}.local` from the `{PATH}\cse571-sp21-project-1` folder.

### 3. Run the packages on the duckiebot.

`dts devel run -H {ROBOT_NAME}.local` from the `{PATH}\cse571-sp21-project-1` folder.

### 4. Visualize what is happening by logging into the robot with the below command.

`dts start_gui_tools --vnc ROBOT_NAME` and open `http://localhost:8087/`
Open the RViz, add `\tf` from the panel like the figure below.
![add_tf](./images/add_tf.png)
*Adding TF to Displays.*

![VNC](./images/novnc.png)
*Example of visualization after adding TF to Displays.*
[Here is the video of what it will look like](https://drive.google.com/file/d/1f476TsCCE0Qec0i8Z3L4r3IU_1kj-CZV/view?usp=sharing)

### 5. Record a screen recording of what you can visualize using RViz and other tools (similar to above).

### (Note) How did the run command know what to run? Change what you are launching with the above run command.

By default, the above command will launch the package(s) indicated in the `./launchers/default.sh` file. 
Be sure to follow these [instructions](https://docs.duckietown.org/daffy/duckietown-robotics-development/out/dt_infrastructure.html) in order to build. You cannot build this on your desktop using the typical `docker build` workflow. 



## TASK-2 Create a physical `map-1` using duckietown supplies

### 1. Create a map in your environment that looks and measures like below. Note that consistent height of the tag center to the base of the stand will avoid noise in the estimation.

![Map1](./images/map1.png)

### 2. Understanding the coordinate frames in the map (both physical as well as what is visualized in the RViz).

In RViz, coordinate frames are colored with x-axis in red, y-axis in green and z-axis in blue. For `map-1`, AprilTAG-31 (AT-31) is the global origin frame. map-1 has two other tags (AT-32 and AT-65) with fixed transformation with respect to the map frame. `my_static_tf` package runs by default from the `./launchers/default.sh` to publish `\tf` tree that you can see in RViz. So overall, you can select 4 transforms (`map`, `april_tag_31`, `april_tag_32`, `april_tag_65`) in RViz to see the map. 

![Map1_RViz](./images/map1_rviz.png)

### 3. Understanding the measurements published in the map frame.

It is important to understand the TF tree.
- When the robot detects an AT, a transform is published linking `april_tag_{id}` to its internal pose as read by the camera `april_tag_cam_{id}`. 
- Another transform links `april_tag_cam_{id}` and `at_{id}_camera_rgb_link` (which is the internal transform of the camera). 
- Another transform links the internal camera to its physical location via `at_{id}_camera_rgb_link` to `at_{id}_camera_link`. 
- Using the design of the robot, we can get the transform between `at_{id}_camera_link` to the base of the robot `at_{id}_base_link`. 

Notice that all of these transforms is specific to a AprilTAG. This is the measurement we provide you when a robot detects an AprilTAG. At the end of the Task 1, you should be able to visualize these transforms in RViz. Move the robot around and see how these transforms change. You can also change the visualization's global frame from \map to any other frame using the "Global options" -> "Fixed frame" in RViz.

### 4. Record what you can visualize in RViz and an image of the physical map created.

## TASK-3 Implement `sensor_fusion_localization` package and test on `map-1`.

After you understand the TF-tree, you can start thinking about our two filtering methods: the Extended Kalman Filter and the Particle Filter. Your task is going to involve modifying the (very rough) skeleton code we provide [here](./packages/sensor_fusion_localization/src/sensor_fusion_node.py) and demonstrating results with the two methods.

We have provided a motion model for you, but you will need to consider how to set the covariance matrix, how to handle the sensor measurements, and, for the Kalman Filter, how to linearize it. The sensor measurements should come in the form of the readings from the AprilTags. Note that the AprilTags provide a 6-DOF pose while we are only attempting to do 3Dof (x, y, theta) localization. Once you have implemented the kalman filter, you should figure out how to visualize the poses. Currently poses are published from the skeleton as Pose2DStamped messages. These are not suported by RViz. You will need to either need to plot your robot's trajectory using alternative tools or change the message type to use one supported by RViz.

Test both of your localization methods on `map-1`. For each method, record a video of the robot navigating and create a simultaneous visualization of the pose estimates. You should lay these out side-by-side in a video editor of your choice and submit it as part of your report.


## TASK-4 Create a physical `map-2` using duckietown supplies

### 1. Create a map in your environment using the instructions provided [here](https://bit.ly/cse571-physical-map).


### 2. Change the static publisher and map file to see the map in RViz with landmarks.

- By default, the `/launchers/default.sh` executes the `viz_node.launch` (for the `map-1`). Change this to `viz_node_2.launch` to visualize `map-2` in RViz.
- By default, the `/launchers/default.sh` executes the `static_at_tf_publisher1.launch`. Change this to `static_at_tf_publisher2.launch` to visualize landmarks (AprilTAGs frames).
- Now build and run the project. You will be able to see the map and the static transforms in RViz as below.

![Map2_RViz](./images/map2_rviz.png)

### 3. Record what you can visualize in RViz and an image of the physical map created.

## TASK-5 Test `sensor_fusion_localization` package on `map-2`.

Similar to Task 4, test your sensor fusion methods on the new map and record videos of your results. These videos should be submitted as part of your report.

## TASK-6 Write up a report

For the submission of this project, you should submit a 3-4 page report.
Here is a list of items that should be included in the report.

**Note: Videos should be uploaded to youtube in the *unlisted* mode and links should be provided in the report.**
1. Group members information: Group name, full individual names, and email.
1. **[20 points]** A video of what you can visualize using RViz or other tools 
   when you build the provided packages on your robot.
1. **[10 points]** A video of RViz or equivalent visualization and (an) image(s) of the physical map for `map-1`.
1. Implementation and experiments on `map-1`:
    1. **[20 points]** Explanation of EKF, PF algorithms, and your implementation details.
    1. **[20 points]** A side-by-side video of visualization and robot navigation on `map-1` with EKF.
    1. **[20 points]** A side-by-side video of visualization and robot navigation on `map-1` with PF.
1. **[10 points]** Video of visualization and (an) image(s) of the physical map for `map-2`.
1. Experiments on `map-2`:
    1. **[40 points]** A side-by-side video of visualization and robot navigation on `map-2` with EKF.
    1. **[40 points]** A side-by-side video of visualization and robot navigation on `map-2` with PF.
1. **[20 points]** Description of what you could do to improve on the results in terms of setup
   (i.e. how might you create a more stable environment for localization) and 
   in terms of methodology (what other methods would you use to do a better job and why)

