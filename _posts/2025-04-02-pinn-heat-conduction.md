---
layout: post
title: Solving 2D Transient Heat Conduction with Physics-Informed Neural Networks
date: 2025-04-02 10:00:00-0400
description: Building a mesh-free PDE solver in PyTorch, validating it against an ANSYS finite-element solution, and what that taught me about scientific machine learning.
tags: pinn  scientific-ml 
categories: 
giscus_comments: false
related_posts: false
toc:
  sidebar: top
---

During my internship,The question I explored was how can surrogate models like Physics informed neural nets be used as surrogate models as a replacement to traditional numerical solvers.  Physics-Informed Neural Networks (PINNs) are a mesh-free way to solve PDEs by baking the governing equations directly into the loss function instead of relying purely on labeled data. I wanted to move past the idea and actually build one, benchmark it rigorously against a trusted numerical solution, and understand where it holds up and where it doesn't.

This post walks through what I built, how I built it, and — more importantly — what the process taught me about scientific machine learning.

## The problem: 2D transient heat conduction

I picked a classic but non-trivial benchmark: transient heat diffusion in a 2D square plate, governed by the heat equation

$$
\frac{\partial T}{\partial t} = \alpha \left( \frac{\partial^2 T}{\partial x^2} + \frac{\partial^2 T}{\partial y^2} \right)
$$

on the domain $$\Omega = \{(x, y, t) \mid x, y \in [0, 1],\ t \in [0, 1]\}$$ (normalized time), with Dirichlet boundary conditions: the bottom wall held at a fixed high temperature, and the top, left, and right walls held at a lower fixed temperature, with a uniform initial condition across the plate. Normalizing the temperatures to a 0–1 range kept the loss landscape well-scaled during training, which turned out to matter more than I initially expected.

To have a credible reference to validate against, I first solved the same problem in **ANSYS Transient Thermal** using a 10,000-node finite-element mesh. That gave me a ground-truth temperature field I could compare the network's predictions against at multiple points in time.

## Why PINNs, and what makes them different

A standard neural network learns a mapping from data alone. A PINN learns a mapping that is *also* constrained to satisfy a differential equation everywhere in the domain, by penalizing the residual of the PDE at randomly sampled points. This blurs the line between data-driven and physics-based modeling: no labeled interior data is required at all — the "supervision" comes from calculus.

That property is what makes PINNs appealing for engineering problems: no expensive mesh generation, and, in principle, better generalization since the model is anchored to physical law rather than pattern-matching alone. It also comes with real costs (I'll get to those), which is exactly why I wanted to benchmark it seriously rather than take the promise at face value.

## Building the network

The architecture is a plain fully-connected network taking `(x, y, t)` as input and predicting temperature `T` as a scalar output, using `tanh` activations (smooth and infinitely differentiable — important, since I need to differentiate the network's output twice with respect to its inputs):

```python
class pinn2d(nn.Module):
    def __init__(self, inp_dim, out_dim, no_layers, no_hidden_units, act=nn.Tanh()):
        super().__init__()
        self.act = act
        self.inp_layer = nn.Linear(inp_dim, no_hidden_units)
        self.hidden_layers = nn.ModuleList(
            [nn.Linear(no_hidden_units, no_hidden_units) for _ in range(no_layers - 1)]
        )
        self.out_layer = nn.Linear(no_hidden_units, out_dim)

    def forward(self, x, y, t):
        data = torch.cat((x, y, t), dim=1)
        out = self.act(self.inp_layer(data))
        for layer in self.hidden_layers:
            out = self.act(layer(out))
        return self.out_layer(out)
```


## Sampling the domain instead of meshing it

Since there's no mesh, the model needs points to evaluate its physics loss at. I used a **Latin Hypercube Sampler** (via `scipy.stats.qmc`) rather than uniform random sampling, because LHS gives much better space-filling coverage of the domain with fewer points — useful when every extra collocation point adds cost to every training step through backpropagation. Concretely, I sampled:

- **~8,000 interior collocation points** `(x, y, t)` where the PDE residual is enforced
- **~800 points per boundary edge** (left, right, top, bottom) where the Dirichlet conditions are enforced
- The same interior `(x, y)` points at `t = 0` for the initial condition

## Encoding the physics: automatic differentiation as the loss

This is the core trick of a PINN, and the part I found most instructive. Instead of discretizing derivatives with finite differences on a grid, I get *exact* derivatives of the network's output with respect to its inputs using PyTorch's `autograd`:

```python
u = pinn(x, y, t)
dudt  = torch.autograd.grad(u, t, grad_outputs=torch.ones_like(t), create_graph=True)[0]
dudx  = torch.autograd.grad(u, x, grad_outputs=torch.ones_like(x), create_graph=True)[0]
dudx2 = torch.autograd.grad(dudx, x, grad_outputs=torch.ones_like(x), create_graph=True)[0]
dudy  = torch.autograd.grad(u, y, grad_outputs=torch.ones_like(y), create_graph=True)[0]
dudy2 = torch.autograd.grad(dudy, y, grad_outputs=torch.ones_like(y), create_graph=True)[0]

pde_loss = (dudt - alpha * (dudx2 + dudy2)).pow(2).mean()
```

`create_graph=True` is what makes second-order differentiation possible — it keeps the computation graph alive so I can differentiate a derivative again. The full loss combines three terms:

$$
\mathcal{L} = \mathcal{L}_{\text{PDE}} + \lambda \cdot \mathcal{L}_{\text{boundary}} + \mathcal{L}_{\text{initial}}
$$

I weighted the boundary loss ($$\lambda = 5$$) more heavily than the PDE residual and initial-condition terms. This was a deliberate, empirical choice: early runs without extra boundary weighting produced solutions that satisfied the interior physics reasonably well but drifted from the prescribed wall temperatures, since the PDE residual term — sampled over thousands of interior points — otherwise dominates the gradient signal. 

## A two-stage optimization strategy

I trained in two phases, which is standard practice for PINNs:

1. **Adam** (5,000 epochs, initial LR 1e-3, exponential decay at rate 0.95 every 100 steps) to get into a good basin of the loss landscape efficiently.
2. **L-BFGS** (a second-order quasi-Newton method, ~3000 iterations) to fine-tune from there.

Adam is great at making fast early progress with noisy, high-dimensional gradients, but PINN loss landscapes are notoriously stiff near convergence — L-BFGS, which uses curvature information, squeezes out the last bit of accuracy far more efficiently than Adam alone. Switching optimizers mid-training rather than committing to just one was one of the clearer lessons from this project: no single optimizer is uniformly best across the whole training trajectory.

## Architecture comparison:

I trained three architectures under identical conditions — a shallow-and-narrow network, a medium one, and a deep-and-wide one — to see how capacity trades off against accuracy for this problem:

| Architecture | Loss after 100 epochs | Validation domain relative error |
|:---|:---:|:---:|
| 4 layers × 50 units | 0.7492 | 0.01456 |
| 5 layers × 100 units | 0.029 | **0.01421** |
| 6 layers × 200 units | 0.40 | 0.01412 |

The 5×100 network turned out to be the sweet spot for this problem: it converged dramatically faster than the other two in early training and landed within a hair of the largest network's final accuracy, at a fraction of the training cost. The 6×200 network's advantage over 5×100 in final relative error was marginal, which shows that **more parameters buys diminishing returns once the network has enough capacity to represent the solution manifold**, and for smooth, low-dimensional PDE solutions like this one, that ceiling is reached quickly.

## Validating against ANSYS

I compared the PINN's predicted temperature field against the ANSYS finite-element solution at five different points in time, computing the average absolute error over the domain at each snapshot.

The PINN tracked the ANSYS solution closely in the interior of the domain across all time steps, with average errors in the low single-digit percent range of the temperature scale. The largest discrepancies consistently showed up **near the boundary at early time**, exactly where the initial condition and the boundary condition are discontinuous with each other (the interior starts at one temperature while the bottom wall is instantaneously held at another). This is a well-known failure mode for PINNs — sharp gradients and discontinuities are hard for a smooth, globally-supported network to represent.

## What this project actually taught me

A few things stuck with me well beyond this specific problem:

- **Loss balancing.** Which term dominates training determines what the network gets "good at" first and getting boundary conditions weighted properly was the difference between a physically meaningful solution and one that merely looked plausible in the interior.
- **Capacity has a ceiling for a given problem.** Throwing more layers and units at a PDE doesn't help once the network can already represent the solution.
- **PINNs trade meshing cost for training cost.** They're compelling when geometry is complex or a mesh is expensive to generate and maintain, but for a well-posed, mesh-friendly problem like this one, a classical FEM solver like ANSYS is still faster and more accurate out of the box. The value proposition of PINNs is clearest in exactly the cases where traditional solvers struggle — irregular domains, inverse problems, and settings where labeled data is scarce but physical laws are known — and this project gave me a much more grounded sense of when that trade-off actually pays off.

## Limitations

One thing this setup does **not** test is generalization across boundary conditions. As built, the boundary temperatures are hardcoded constants inside the loss function rather than inputs to the network — so the trained model only ever knows how to solve the one boundary-value problem it was trained on.



*Full implementation (PyTorch, `autograd`-based PDE residuals, LHS sampling, Adam + L-BFGS training) available at [github](https://github.com/ManojBhatta/pinn)*