---
layout: distill
description: >-
  Follow my steps to learn how to publish a pointcloud using ROS2
  publisher-subscriber and visualise it in Rviz2
tags: ROS2 azure_kinect pointcloud sensors
giscus_comments: true
date: 2024-02-11T00:00:00.000Z
featured: true
authors:
  - name: Karlym Nam
toc:
  - name: Microsoft Azure Kinect
  - name: Github Repository + Other Resources
  - name: Repo Review
  - name: Time for a custom code
  - name: Result
_styles: |
  .fake-img {
    background: #bbb;
    border: 1px solid rgba(0, 0, 0, 0.1);
    box-shadow: 0 0px 4px rgba(0, 0, 0, 0.1);
    margin-bottom: 12px;
  } .fake-img p {
    font-family: monospace;
    color: white;
    text-align: left;
    margin: 12px 0;
    text-align: center;
    font-size: 16px;
  }
images:
  compare: true
  slider: true
---
{% include figure.liquid path="assets/img/1_azure_kinect/final.png" class="img-fluid rounded z-depth-1" zoomable=true%}

I publish my own representation of pointcloud to ROS2 and visualise it in RVIZ. ROS2 Driver also publishes a coloured pointcloud which merges pointcloud + rgb camera feed but I found some serious latency. This one should be pretty simple.
## Microsoft Azure Kinect
We are using Microsoft Azure Kinect today. I'm running this on **Ubuntu 22.04** and **ROS2 humble** (also tested it on iron). You might want to also install **k4aviewer** to check the connection to the kinect. There are several settings for depth camera and rgb camera to play around. 

<swiper-container keyboard="true" navigation="true" pagination="true" pagination-clickable="true" pagination-dynamic-bullets="true" rewind="true">
  <swiper-slide>{% include figure.liquid path="assets/img/1_azure_kinect/NFOV_binned.png" class="img-fluid rounded z-depth-1" %}</swiper-slide>
  <swiper-slide>{% include figure.liquid path="assets/img/1_azure_kinect/NFOV_unbinned.png" class="img-fluid rounded z-depth-1" %}</swiper-slide>
  <swiper-slide>{% include figure.liquid path="assets/img/1_azure_kinect/WFOV_binned.png" class="img-fluid rounded z-depth-1" %}</swiper-slide>
  <swiper-slide>{% include figure.liquid path="assets/img/1_azure_kinect/WFOV_unbinned.png" class="img-fluid rounded z-depth-1" %}</swiper-slide>
</swiper-container>

Unfortunately, I am running everything on my laptop without any GPU so I can't visualise 3d pointcloud just on k4aviewer. Hopefully I can in Rviz2!
## Github Repository + Other Resources
- [Azure Kinect ROS driver](https://github.com/microsoft/Azure_Kinect_ROS_Driver)
-- I used Humble branch
-- ARE YOU STRUGGLING TO RUN THE GIT REPO ABOVE? BECAUSE I DID TOO. 

## Repo Reivew
Upon running the launch file for Azure Kinect ROS driver, I can already access several information such as camera parameters, rgb camera feed, depth camera feed, various transformations and even pointcloud. 

```
ros2 topic list

/clicked_point
/depth/camera_info
/depth/image_raw
/depth_points
/depth_to_rgb/camera_info
/depth_to_rgb/image_raw
/goal_pose
/imu
/initialpose
/ir/camera_info
/ir/image_raw
/joint_states
/parameter_events
/points2
/rgb/camera_info
/rgb/image_raw
/rgb_to_depth/camera_info
/rgb_to_depth/image_raw
/robot_description
/rosout
/tf
/tf_static

```

Once I open rviz2 and select point_cloud, MY LAPTOP IS STRUGGLING. 

{% include figure.liquid path="assets/img/1_azure_kinect/default_pointcloud.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Let's write some easy python code to access the raw depth data and create a simpler representation of pointcloud. Then we can do stuff like object detection, pointcloud segmentation, any calculations for your applications.. etc
## Time for a Custom Code!
We make a new ROS2 package which contains a subscriber (subscribe to the depth and camera info) and a publisher (anything you want to publish + simpler rviz representation of the pointcloud).

My workspace structure for now:
```
ws_pointcloud
	\src
```

Then we are going to create a ROS2 Python package with some dependencies inside `ws_pointcloud/src`. I am going to call my package ros2_pointcloud and a node inside called publish_pointcloud_only. Starting with four dependencies including rclcpp, sensor_msgs, cv_bridge, and numpy
```
ros2 pkg create --build-type ament_python ros2_pointcloud --node simple_pointcloud --dependencies rclcpp sensor_msgs cv_bridge numpy
```

Navigate to the code file inside `ws_pointcloud/src/ros2_pointcloud/ros2_pointcloud/simple_pointcloud.py` and start modifying!

```
import rclpy
from rclpy.node import Node

import numpy as np
import message_filters

from sensor_msgs_py import point_cloud2
from sensor_msgs.msg import PointCloud2, PointField, CameraInfo, Image
from cv_bridge import CvBridge

class SimplePointCloudNode(Node):
    def __init__(self):
        super().__init__('simple_pointcloud_node')
        
        self.depth_cam_sub = message_filters.Subscriber(self, Image, "/depth/image_raw")
        self.cam_info_sub = message_filters.Subscriber(self, CameraInfo, "/depth/camera_info")
        
        self.point_pub = self.create_publisher(
            PointCloud2,
            '/depth_points',
            10
        )

        self.mask_pub = self.create_publisher(
            PointCloud2,
            '/mask_points',
            10
        )

        ts = message_filters.ApproximateTimeSynchronizer([self.depth_cam_sub, self.cam_info_sub], 10, 0.5)
        ts.registerCallback(self.cloudCallback)

        self.bridge = CvBridge()
        
        self.skip = 8

        
    def cloudCallback(self, image_msg, info_msg):
        height = image_msg.height
        width = image_msg.width

        data_depth = np.array(self.bridge.imgmsg_to_cv2(image_msg, desired_encoding='passthrough')).reshape(height, width)

        point_cloud =[]

        for x in range(0, height, self.skip):
            for y in range(0, width, self.skip):
                depth = data_depth[x][y]
                point_cloud.append(self.depthToPointCloudPos(info_msg, x, y, depth))

        self.publishCloud(point_cloud, image_msg, self.point_pub)

    def depthToPointCloudPos(self, info_msg, x, y, d):
    
        # We will use this later
        fx = info_msg.k[0]
        fy = info_msg.k[4]
        cx = info_msg.k[2]
        cy = info_msg.k[5]

        point_x = (x - cx) * d / fx
        point_y = (y - cy) * d / fy
        point_z = d
        return x, y, d

    def publishCloud(self, points, img_msg, publisher):
        if len(points) > 0:
            header = img_msg.header
            cloud = point_cloud2.create_cloud_xyz32(header, points)
            publisher.publish(cloud)
            self.get_logger().info("Published matching points as PointCloud2")

def main(args=None):
    rclpy.init(args=args)
    node = SimplePointCloudNode()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()

if __name__ == '__main__':
    main()

```


Sometimes, it is tempting to copy and paste the code so that it works but most of times you might have to tweak a few things so it works on your application. Then a bit of understanding of what's going on is important. 

## Result
You might want to play around with Rviz marker setting so it appears. I had to change my marker size to 1 and change the reference frame to 'depth_camera_link' since I am working in pixel coordinates.

{% include figure.liquid path="assets/img/1_azure_kinect/result.png" class="img-fluid rounded z-depth-1" zoomable=true%}

**Comments**
- My laptop isn't lagging as much although there is some latency.

**Where do we go from here?**
- You can read about how I implemented pointcloud segmentation using object detection.
- You might want to add your own computation and publish the result to something..
