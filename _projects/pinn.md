---
layout: page
title: PINN
description: Implementation of Physics Informed Neural Network for 2d heat conduction
img: assets/img/pinn.png
importance: 3
category: work
related_publications: true
---

## Overview
The goal of the project was to develop a PINN as a surrogate model for simulating 2D transient heat conduction in a square plate, using only the governing PDE as supervision — no labeled training data.

## Implementation details
The network is trained only on the physics and PDE loss; no data points are used. The PDE residual is incorporated directly into the loss function via `autograd`, and the network is trained through backpropagation in a two-stage optimization: first for 5000 epochs using the Adam optimizer, then fine-tuned using L-BFGS. The architecture is a simple MLP, and performance was benchmarked across networks of varying depth and width.

{% include figure.liquid loading="eager" path="assets/img/projects/pinn/architecture.png" class="img-fluid rounded z-depth-1" %}
<div class="caption">
    Overview of the PINN architecture.
</div>

## Results
The PINN's predicted temperature field was validated against a 10,000-node ANSYS finite-element solution at multiple points in time. The average error across the domain was about 2%, with the largest discrepancies appearing near the boundary at early time — a result of the discontinuous, instantaneous jump between the bottom boundary held at high temperature and the other boundaries held at zero.

{% include figure.liquid loading="eager" path="assets/img/projects/pinn/training_loss.png" class="img-fluid rounded z-depth-1" %}
<div class="caption">
    Training loss across the two-stage optimization: Adam for the first 5000 epochs, followed by L-BFGS fine-tuning.
</div>

{% include figure.liquid loading="eager" path="assets/img/projects/pinn/pinn_output.png" class="img-fluid rounded z-depth-1" %}
<div class="caption">
    PINN-predicted temperature field.
</div>

{% include figure.liquid loading="eager" path="assets/img/projects/pinn/ansys_output.png" class="img-fluid rounded z-depth-1" %}
<div class="caption">
    ANSYS finite-element reference solution.
</div>

{% include figure.liquid loading="eager" path="assets/img/projects/pinn/error.png" class="img-fluid rounded z-depth-1" %}
<div class="caption">
    Error between the PINN prediction and the ANSYS solution across the domain.
</div>

## Learning
Weighting the boundary loss more heavily than the PDE residual was key to getting a physically meaningful solution rather than one that only looked plausible in the interior. Switching from Adam to L-BFGS partway through training also mattered: Adam makes fast early progress on the noisy, high-dimensional loss, while L-BFGS's curvature information squeezes out the last bit of accuracy near convergence. More broadly, capacity had a ceiling for this problem — a 5-layer x 100-unit network reached accuracy close to a much larger one at a fraction of the training cost.

More details in [blog]() and Full implementation available on [GitHub](https://github.com/ManojBhatta/pinn).