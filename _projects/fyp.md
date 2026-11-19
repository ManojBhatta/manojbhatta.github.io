---
layout: page
title: Autonomous Navigation
description: Designed and developed an AGV capable of navigating unknown environments using LiDAR-based SLAM and Nav2.
img: assets/img/fyp.png
importance: 1
category: work
related_publications: false
mermaid:
  enabled: true
  zoomable: false
---

<div class="row justify-content-center">
  <div class="col-sm-10 col-md-7">
    <video width="100%" controls autoplay loop muted playsinline preload="auto" class="img-fluid rounded z-depth-1">
      <source src="{{ '/assets/video/projects/fyp/demo_vid.mp4' | relative_url }}" type="video/mp4">
      Your browser does not support the video tag.
    </video>
    <div class="caption">
      The AGV autonomously mapping and navigating an outdoor environment.
    </div>
  </div>
</div>

## Overview

As part of our final year project, my team — Biswash Khatiwada, Susil Chhetri, Balkrishna Poudel, and I — designed and built an
Autonomous Ground Vehicle (AGV) capable of navigating unknown outdoor environments using LiDAR-based SLAM and the Nav2 navigation
stack.

| Compute                                     | Perception     | Low-level control               | Software                     |
| ------------------------------------------- | -------------- | ------------------------------- | ---------------------------- |
| Onboard laptop, ROS 2 Humble (Ubuntu 22.04) | 3D LiDAR + IMU | STM32 microcontroller over UART | Fast-LIO, SLAM Toolbox, Nav2 |

## My Contribution

- **Simulation:** Set up an accurate Gazebo simulation of the robot for testing SLAM and navigation before deployment on real
  hardware.
- **Embedded control:** Developed the low-level hardware interface on an STM32 microcontroller responsible for converting ROS 2
  `Twist` commands into motor commands, including motor kinematics and PID control.
- **Communication:** Wrote the ROS 2 node handling UART communication between the onboard PC and the STM32, including packet
  framing and robust serial data handling.
- **Navigation:** Tuned and tested Nav2 parameters for reliable path planning and obstacle avoidance.

## Implementation Approach and Details

### Hardware

The chassis was built by welding together the frames of two Segways to form the base of the AGV. An onboard laptop running ROS 2
Humble on Ubuntu 22.04 served as the robot's main compute, handling perception, localization, mapping, and navigation. A 3D LiDAR
provided the primary perception input, and an STM32 microcontroller handled all low-level motor control, communicating with the ROS
2 stack over UART.

<style>
  .equal-ratio img { aspect-ratio: 4 / 3; object-fit: contain; background: #fff; }
  pre.mermaid { text-align: center; }
  .arch-legend { display: flex; flex-wrap: wrap; justify-content: center; gap: 0.4rem 1.2rem; font-size: 0.9rem; margin-top: -0.5rem; }
  .arch-legend i { display: inline-block; width: 0.9rem; height: 0.9rem; border: 1.5px solid; border-radius: 3px; margin-right: 0.4rem; vertical-align: -0.1rem; }
</style>
<div class="row align-items-center g-3 equal-ratio">
    <div class="col-sm-6">
        {% include figure.liquid loading="eager" path="assets/img/projects/fyp/electronics_setup.png" title="Electronics setup schematic" class="img-fluid rounded z-depth-1 w-100" %}
    </div>
    <div class="col-sm-6">
        {% include figure.liquid loading="eager" path="assets/img/projects/fyp/hardware_setup.png" title="Real hardware setup" class="img-fluid rounded z-depth-1 w-100" %}
    </div>
</div>
<div class="caption">
    Electronics schematic (left) and the assembled hardware platform (right).
</div>

### System architecture

The overall system architecture is shown below:

```mermaid
%%{init: {"themeVariables": {"fontSize": "18px"}, "flowchart": {"rankSpacing": 40, "nodeSpacing": 40}}}%%
flowchart TB
    subgraph SENSORS[" "]
        LIDAR["LiDAR<br/>3D point cloud"]
        IMU["IMU<br/>Orientation & accel"]
    end

    subgraph PERCEPTION[" "]
        FASTLIO["Fast-LIO<br/>LiDAR-inertial odometry"]
        SCANCONV["Laser Scan Conversion<br/>3D → 2D, height filtering"]
    end

    subgraph AUTONOMY[" "]
        SLAM["SLAM Toolbox<br/>Occupancy grid mapping"]
        NAV2["Nav2<br/>Path planning & control"]
    end

    subgraph EMBEDDED[" "]
        MCU["Microcontroller<br/>cmd_vel → motor commands"]
        DRV_L["Motor Driver L<br/>Left side"]
        DRV_R["Motor Driver R<br/>Right side"]
        MOT_L["Drive + Steering Motors<br/>Left side"]
        MOT_R["Drive + Steering Motors<br/>Right side"]
    end

    LIDAR --> FASTLIO
    IMU --> FASTLIO
    LIDAR --> SCANCONV
    FASTLIO -- "Point Cloud" --> SCANCONV
    FASTLIO -- "Odometry" --> NAV2
    SCANCONV -- "2D Scan" --> SLAM
    SLAM -- "Map" --> NAV2
    NAV2 <-- "cmd_vel (UART)" --> MCU

    MCU --> DRV_R
    MCU --> DRV_L
    DRV_R --> MOT_R
    DRV_L --> MOT_L

    classDef sensor fill:#E6F1FB,stroke:#185FA5,color:#042C53
    classDef perception fill:#EEEDFE,stroke:#534AB7,color:#26215C
    classDef autonomy fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    classDef embedded fill:#FAECE7,stroke:#993C1D,color:#4A1B0C

    class LIDAR,IMU sensor
    class FASTLIO,SCANCONV perception
    class SLAM,NAV2 autonomy
    class MCU,DRV_L,DRV_R,MOT_L,MOT_R embedded

    style SENSORS fill:#185FA5,fill-opacity:0.08,stroke:#185FA5
    style PERCEPTION fill:#534AB7,fill-opacity:0.08,stroke:#534AB7
    style AUTONOMY fill:#0F6E56,fill-opacity:0.08,stroke:#0F6E56
    style EMBEDDED fill:#993C1D,fill-opacity:0.08,stroke:#993C1D
```

<div class="arch-legend">
  <span><i style="background:#E6F1FB;border-color:#185FA5"></i>Sensors</span>
  <span><i style="background:#EEEDFE;border-color:#534AB7"></i>Perception &amp; Localization (Host PC)</span>
  <span><i style="background:#E1F5EE;border-color:#0F6E56"></i>Autonomy Stack (ROS 2, onboard compute)</span>
  <span><i style="background:#FAECE7;border-color:#993C1D"></i>Low-Level Control (Embedded)</span>
</div>

### Perception and mapping

Fast-LIO was used to fuse the 3D LiDAR and IMU data into a real-time LiDAR-inertial odometry estimate, which grounded the robot's
pose during mapping and navigation. The resulting 3D point cloud was height-filtered and projected down to a 2D laser scan, which
SLAM Toolbox consumed to build the occupancy grid map used for localization and path planning.

### Control and navigation

On the navigation side, motor kinematics and PID control were implemented directly on the STM32, which received `cmd_vel`
(`Twist`) commands from the ROS 2 side over UART and converted them into per-wheel motor commands. Within Nav2, the default plugins
were used for planning and control:

| Role                       | Nav2 plugin        | Notes                                            |
| -------------------------- | ------------------ | ------------------------------------------------ |
| Local planner / controller | **DWB controller** | Dynamic Window Approach-based                    |
| Global planner             | **NavFn planner**  | Computes global paths using Dijkstra's algorithm |

Only a subset of parameters in the `nav2_params.yaml` file were tuned from their defaults — mainly costmap inflation, controller
velocity/acceleration limits, and goal tolerances — based on testing.

## Results

Since localization relied solely on 2D laser scan matching (no additional sensor fusion for pose correction), goal-reaching
accuracy varied noticeably with environment. Indoors, where structured features like walls and corners gave the scan matcher
plenty to work with, the robot reached its goal with a position error of about 20 cm. Outdoors, sparser and less consistent
features led to greater drift in localization, with goal position errors of around 50 cm.

<div class="row align-items-center g-3">
    <div class="col-md-8">
        {% include figure.liquid loading="eager" path="assets/img/projects/fyp/goal_error.png" title="Goal position error comparison" class="img-fluid rounded z-depth-1 w-100" %}
    </div>
    <div class="col-md-4">
        <table class="table table-sm mb-0">
            <thead><tr><th>Environment</th><th>Goal error</th></tr></thead>
            <tbody>
                <tr><td>Indoor</td><td><strong>~20 cm</strong></td></tr>
                <tr><td>Outdoor</td><td><strong>~50 cm</strong></td></tr>
            </tbody>
        </table>
    </div>
</div>
<div class="caption">
    Difference between the commanded goal and the robot's actual stopping position, comparing indoor and outdoor runs
    (the robot was supposed to stop inside the four corners).
</div>

## Learnings

This project was a deep dive into the full autonomy stack including the ROS 2 framework, SLAM Toolbox and LiDAR-inertial SLAM, path
planning and navigation with Nav2, simulating the robot in Gazebo, and building and debugging the hardware and embedded electronics
end-to-end. Beyond the technical skills, it was also a great exercise in integrating perception, planning, and control into one
working system and of course it was a lot of fun to build (and ride the robot).

---

**Repository:** [github](https://github.com/manojbhatta/bail_gada) · **Final Thesis:** [pdf]({{ '/assets/pdf/fyp.pdf' | relative_url }})
