2025.1.2
This is for an unofficial unitree go2 sdk, original project is https://github.com/abizovnuralem/go2_ros2_sdk
Bug 1:
  The cost map especially global cost map has huge difference with local costmap and map.
Solution 1:
  Change nav2_parameters.yaml, "global_frame:odom"->"global_frame:map", in "amcl","bt_navigator","global_costmap" section.
Reason 1: 
  Original project setting these frames to odom will cause the fact that all localization node whether in SLAMtoolbox or Nav2 are useless. And this will make all data(Passable area，obstacle....) in " odom" coordinates. And with running time increases， error between odom and map will accummate but no any node can erase this. Because localization node shoud input /scan and output the right /tf between local_frame and global_frame to make sure that costmap can match the global frame:map. But right now, local_frame=global_frame=odom. So the costmap especially global costmap will be very different with realy world.
  
Bug 2:
  Nav2 can not continuously work, and cost map especially global cost map will stop to update just one or two mins after connection build. There is an error raised:
  "Extrapolation Error looking up target frame: Lookup would require extrapolation into the past. Requested time 1735567638.139486 but the earliest data is at time 1735567638.358660, when looking up transform from frame [base_link] to frame [map]"
  
Solution 2:
  Change go2_driver_node.py, use BEST_EFFORT in qos of lidar
"lidar_qos = QoSProfile(
    reliability=QoSReliabilityPolicy.BEST_EFFORT,  # 改用best effort
    durability=QoSDurabilityPolicy.VOLATILE,       # 添加volatile持久性
    history=QoSHistoryPolicy.KEEP_LAST,
    depth=100                                      # 增加缓存深度
)"
Reason 2:
  This problem took four days to debug!!!!Really tiny but has huge effect. Firstly, you should know what is the workflow of Nav2(Really important). You can search online, I will only explain the input and output of it. Its input is /scan,costmaps, /tf(baselink->odom,odom->map),and output is /plan and velocity control command.  It will use laserscan in /scan and costmaps for localization and output the the tf(odom->map) to erase the accumulated error caused by odom to make robot's position correct. That's why it wants the tf(base_link->map=base->odom*odom->map). 
  But it will fail, if any link(base_link->odom,odom->map) break. Base_link-->odom is published by robot, so it's totally fine. The problem could only be odom->map which is published by localization node(default amcl). 
  And two input:/scan,costmaps of amcl. I can see costmaps by rviz2, there is no problem. So I monitor the /scan. Yes, it is.
  /Scan should continuously publish, but it will stop sometimes and won't recover. The only one input of /scan is point_cloud2.
  OK, right now, you know why we change the point_cloud2's quality of service. Default setting is ”realiable“ like promising "I must deliver all packages", but when overloaded, the entire service crashes，BEST_EFFORT is like "I'll try my best to deliver, will drop if necessary", service keeps running even if some packages are dropped.






  
Bug 3: Nav2 can't plan a rotation path to robot.
Solution 3:
  Add a RotationShimController to nav2_parameters.yaml:
  "FollowPath:
      plugin: "nav2_rotation_shim_controller::RotationShimController"
      primary_controller: "dwb_core::DWBLocalPlanner"
  "
Reason 3:
  This controller is design for rotation, it will make robot firstly rotate to the path and then let primary controller to plan the next step.

  
       
  
  



