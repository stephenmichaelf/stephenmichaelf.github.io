---
layout: post
title: Learning - Depth Estimation
---

As part of my goal to learn a bunch of new things, I am starting with `depth estimation`. I will either do 1/5/10 hours of learning, depending on how it goes! This post will keep updating.

Time Spent
2024-04-27 - 2:15-3:15 // 1 hour

### What is depth estimation?

Depth estimation is measuring the distance of each pixel in an image (or video) relative to the camera that took it. This can come from either one cameera (monocular) or multiple (stereo).

This is useful for many use cases, like self driving cars and robotics, drones, and more. The information can be fed into Machine Learning algorithms (TODO: How do the intermediary steps look between an image coming in and an action being take. The image is converted to ... X then that is fed to the network? That must not be the case for depth estimation I think).




### How it works

Looks at things like:

- Relative object size
- What else?
- 


### High Quality Monocular Depth Estimation via Transfer Learning

https://arxiv.org/pdf/1812.11941v2


### Depth estimation from images vs. video



### Alternative methods

What else can we use? What other type of sensors are there to estimate depth and how does their efficacy compare to using images/video?

Is there a standard benchmark that people use?

What is state of the art for: self driving cars, robotics, etc.

LiDAR
Laser Range Scanners
Depth Camera
Radar
Sonar




### Real-time depth estimation

How does this work? What sensors are needed?


### MIDAS Neural Networks

What is it? How does it work? Examples

### Related Concept: Structure From Motion

What is this? How is it different? What are the use cases?

### Monocular vs. multi-ocular?

Who uses what. What are pro/cons of each? How do you consolidate the two? How does it affect machine learning?

### Elon Other

Active Optical, for wavelength that os occlusion penetrating - see through fog/rain/dust, 4mm wavelength

### Other Concepts

Stereo vision methods
Scene restructuring
Motion planning



### Measures of effectiveness

#### Scale Invariant Error

https://arxiv.org/abs/1406.2283

### Robotics Concepts from Anduril

radar design, signal processing, probability and statistics, and algorithm development
radar verification and validation, including test design and results documentation
phased array and MIMO technologies for radar systems, with expertise in AESA/PESA system architectures

Familiarity with the evaluation of advanced radar and electronic warfare (EW) technologies, with an emphasis on airborne surveillance radars performing moving target indication (MTI), synthetic aperture radar (SAR), inverse synthetic aperture radar (ISAR), high range resolution (HRR) target imaging

Understanding of antenna theory and familiarity with the design and development of RF subsystems for Radar, EW, communications
Proficiency with typical commercial RF system modelling tools, such as Mathworks Matlab Phased Array System and RF Toolboxes, Mathworks Simulink, AGI STK

Understanding of digital filtering, linear algebra, stochastic processes, estimation theory, and detection theory
computer vision, sensor fusion, SLAM, motion planning, machine learning

Gazebo, Unity, or Unreal Engine

Rust programming





### Links to read

https://towardsdatascience.com/depth-estimation-1-basics-and-intuition-86f2c9538cd1
https://huggingface.co/tasks/depth-estimation
https://paperswithcode.com/task/depth-estimation
https://medium.com/@patriciogv/the-state-of-the-art-of-depth-estimation-from-single-images-9e245d51a315
https://paperswithcode.com/paper/high-quality-monocular-depth-estimation-via
https://betterprogramming.pub/my-journey-with-depth-estimation-8c53ab25cd8b
https://cdn.intechopen.com/pdfs/37767/InTech-Depth_estimation_an_introduction.pdf


This looks interesting - https://arxiv.org/abs/2111.08600


### Videos

https://www.youtube.com/watch?v=kVoIIsdoQR4
https://www.youtube.com/watch?v=BFdWsJs6z4c









