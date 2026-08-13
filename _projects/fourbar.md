---
layout: page
title: Four-bar mechanism synthesis
description: Calculating four bar parameters to trace a given path
img: assets/img/fourbar.png
importance: 3
category: work
related_publications: true
---

The project is a python implementation to synthesizes a four-bar linkage's nine design parameters so its coupler point traces a given target path, using differential evolution and a circular-proximity objective function.

Built with **Sumit Thakur Barahi** and **Benzeena Dhakal** for the Samsung Innovation Campus program at Pulchowk Campus, IOE, Tribhuvan University.

### Overview

Given a target curve as a set of precision points, the goal is to find the nine linkage parameters,

- link lengths $r_1$–$r_4$,
- coupler offset $r_5, \beta$, and
- fixed-pivot pose $x_A, y_A, \alpha$

so a coupler-plane point traces it as closely as possible. Analytical methods cap out at nine precision points; beyond that, synthesis is framed as global optimization instead, solved here with SciPy's differential evolution.

{% include figure.liquid path="assets/img/projects/fourbar/9vars.png" title="Nine design variables of a path-generator four-bar linkage" class="img-fluid rounded z-depth-1" %}

<div class="caption">
    The nine design variables of a path-generator four-bar linkage.
</div>

### Implementation

Only 4 variables — pivot $(x_A, y_A)$, coupler length $r_3$, offset angle $\beta$ — are optimized directly; the rest are recovered analytically each evaluation:

- **Circular Proximity Function (CPF):** scores how closely link 4's traced rocker points fit a circle, since a valid rocker arm sweeps an arc. This keeps the search 4-dimensional regardless of point count (Kafash & Nahvi, 2017).
- **Grashof penalty:** penalizes linkages that fail Grashof's condition or where the input link isn't shortest, ruling out non-crank solutions.
- **Differential evolution:** searches the 4D space via `scipy.optimize.differential_evolution`.
- **Animation:** Matplotlib animates the resulting crank-rocker motion against the target curve.

### Results and learnings

<div class="row">
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/projects/fourbar/mech1.gif" title="Crank-rocker mechanism tracing a looped path" class="img-fluid rounded z-depth-1" %}
    </div>
    <div class="col-sm mt-3 mt-md-0">
        {% include figure.liquid path="assets/img/projects/fourbar/mech2.gif" title="Crank-rocker mechanism tracing a figure-eight path" class="img-fluid rounded z-depth-1" %}
    </div>
</div>
<div class="caption">
    Optimized linkages tracing two randomly generated target curves.
</div>

Run for 50 generations on two test curves of increasing complexity, the optimizer converged on valid crank-rocker linkages that visibly track each target.

- Reformulating around CPF instead of raw tracking error is what keeps the search 4D regardless of point count — the main efficiency win.
- Without the Grashof/shortest-link penalty, DE often converged on double-rockers: good least-squares fits that can't actually be driven.
- Each precision point admits two valid coupler circuits (open/crossed); both are evaluated and the better one kept.

Code: [GitHub](#) (add link).
