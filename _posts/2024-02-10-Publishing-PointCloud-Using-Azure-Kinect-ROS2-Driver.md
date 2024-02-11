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
  - name: Github Repository and Other Resources
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
  slider: true
---
{% include figure.liquid path="assets/img/1_azure_kinect/final.png" class="img-fluid rounded z-depth-1" %}
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
-[Azure Kinect ROS driver](https://github.com/microsoft/Azure_Kinect_ROS_Driver)

- I used Humble branch
- ARE YOU STRUGGLING TO RUN THE GIT REPO ABOVE? BECAUSE I DID TOO. 

## Code Reivew
Upon running the launch file for Azure Kinect ROS driver, I can already access several information such as rgb camera feed, depth camera feed and even pointcloud. Once I open rviz2 and select point_cloud, MY LAPTOP IS STRUGGLING. 

{% include figure.liquid path="assets/img/1_azure_kinect/default_pointcloud.png" class="img-fluid rounded z-depth-1" %}

## Time for a Custom Code!
## Result
