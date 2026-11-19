---
layout: page
title: Open-Vocabulary Navigation
description: A robot maps a room and then goes to any object you name in plain words, such as "lamp" or "sofa"
img: assets/img/ovnav.png
importance: 2
category: work
related_publications: false
mermaid:
  enabled: true
  zoomable: false
---

<div class="row justify-content-center">
  <div class="col-sm-12 col-md-10">
    <video width="100%" controls autoplay loop muted playsinline class="img-fluid rounded z-depth-1">
      <source src="{{ '/assets/video/projects/ov_nav/nav.mp4' | relative_url }}" type="video/mp4">
      Your browser does not support the video tag.
    </video>
    <div class="caption">The robot gets a text query. It plans a path and drives to the object.</div>
  </div>
</div>

## Overview

Most robot maps know only a fixed list of objects. This map stores a CLIP feature in each cell, not a label. Thus, you can ask for
any object in plain words, and the robot finds it with no new training.

The method comes from [VLMaps](https://vlmaps.github.io/) (ICRA 2023). I built it again for **ROS 2 Jazzy + Gazebo Harmonic**, not
the original Habitat simulator.

For the full setup and usage, refer to the [repository](https://github.com/manojbhatta/ov_nav).

## System

```mermaid
flowchart TD
    subgraph BUILD["1 · Build the map (once)"]
        direction LR
        R["Robot in Gazebo<br/>RGB-D camera + pose"] --> L["LSeg<br/>one CLIP feature per pixel"] --> G["2D grid, 5 cm cells<br/>features averaged per cell"]
    end
    subgraph USE["2 · Use the map (each query)"]
        direction LR
        Q["Text query<br/>'lamp'"] --> T[CLIP text encoder] --> S["Cosine similarity<br/>with every cell"] --> P["Goal pose<br/>best cells, free of obstacles"] --> N["Nav2<br/>robot drives to goal"]
    end
    G --> S
```

## The map

<div class="row justify-content-center align-items-center g-2">
    <div class="col-6 col-md-3">
        {% include figure.liquid loading="eager" path="assets/img/gz_env.png" title="Gazebo environment" class="img-fluid rounded z-depth-1 w-100" %}
    </div>
    <div class="col-6 col-md-3">
        {% include figure.liquid loading="eager" path="assets/img/color_top_down.png" title="Color top-down map" class="img-fluid rounded z-depth-1 w-100" %}
    </div>
    <div class="col-6 col-md-3">
        {% include figure.liquid loading="eager" path="assets/img/obstacle_map.png" title="Obstacle map" class="img-fluid rounded z-depth-1 w-100" %}
    </div>
    <div class="col-6 col-md-3">
        {% include figure.liquid loading="eager" path="assets/img/vlmap_category.png" title="Map with semantic categories" class="img-fluid rounded z-depth-1 w-100" %}
    </div>
</div>
<div class="caption">Gazebo living room · color top-down map · obstacle map · map with semantic categories.</div>

## Ask with words

Each query gives a similarity heatmap over the full map. Yellow is high similarity. The red mark is the goal.

<div class="row justify-content-center g-2">
    <div class="col-sm-6">
        {% include figure.liquid loading="lazy" path="assets/img/projects/ovnav/query_coffee_table.png" title="Query: coffee table" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-6">
        {% include figure.liquid loading="lazy" path="assets/img/projects/ovnav/query_sofa.png" title="Query: sofa" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-6">
        {% include figure.liquid loading="lazy" path="assets/img/projects/ovnav/query_rug.png" title="Query: rug" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm-6">
        {% include figure.liquid loading="lazy" path="assets/img/projects/ovnav/query_lamp.png" title="Query: lamp" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">Queries: coffee table · sofa · rug · lamp. The same map answers all four.</div>

## Why it works

CLIP puts images and text in the same feature space. Thus, a text query can point to the image areas that match it.

<div class="row justify-content-center">
    <div class="col-12">
        {% include figure.liquid loading="lazy" path="assets/img/clip_similarity_output.png" title="CLIP patch similarity for a text query" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">CLIP patch similarity for the query "rug": RGB · depth · 14x14 similarity grid · heatmap on the image.</div>

## Lessons

- **Dense features are the hard part.** Raw CLIP patches are coarse and noisy. LSeg gives per-pixel features that are good for a
  map.
- **Grid settings change the map.** Cell size, depth sampling and camera height all move the floor and obstacle edges.
- **Sync the sensors yourself.** The approximate-time synchronizer stalled with Gazebo QoS. A simple closest-timestamp cache
  works and is easy to debug.
- **Align the frames.** The map starts at the robot start pose, but Nav2 uses the `map` frame. A static `tf2` transform is the
  correct fix.

---

**Repository:** [github.com/manojbhatta/ov_nav](https://github.com/manojbhatta/ov_nav) · **Based on:**
[VLMaps](https://vlmaps.github.io/) (Chen et al., ICRA 2023), LSeg (Li et al., CVPR 2022), Nav2, SLAM Toolbox and Gazebo.
