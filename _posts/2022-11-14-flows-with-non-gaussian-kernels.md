---
layout: post
title: "[WIP] Flows with non-gaussian kernels"
toc: true
---

<div style="display:none">
  $
    \DeclareMathOperator*{\argmin}{argmin}
  $
</div>

\[**Under construction. If you see this it's probably because I shared it directly with you.**\]

(Thanks to Tero Karras for the pointer to [Luhman & Luhman's paper](https://arxiv.org/abs/2207.04316))

In my previous post we see that we can derive some fun identities based around diffusion kernels.
It turns out we can follow a similar logic for arbitrary kernels. For exponential family kernels, we get some simple
plug-in models.

# tl;dr

#### In general

A score model can be constructed from conditional distributions as:

$$\dot{x} \propto \mathbb{E}_{p(x_0 \vert x; t)}\left[ \nabla \ln p(x \vert x_0; t) \right]$$

Its Jacobian is

$$
\nabla \dot{x} \propto \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ H \ln p(x \vert x_0; t) \right]
+ \mathbb{V}_{x_0 \sim p(x_0 \vert x; t)} \left[\nabla \ln p(x \vert x_0; t) \right]
$$

#### Exponential family

For exponential family models, it's

$$\begin{alignat}{2}
\dot{x} & \propto  \nabla \ln h(x) + \nabla T(x) \cdot \mathbb{E}\left[ \eta(\theta_{x_0, t}) \vert x \right]
\end{alignat}$$

and

$$\begin{alignat}{2}
\nabla \dot{x} \propto & \; H \ln h(x) + D_{jk} T_i(x) \cdot \mathbb{E} \left[\eta_i(\theta_{x_0, t}) \,\vert\, x \right] \\ 
 & + \nabla T(x) \cdot \mathbb{V} \left[\eta(\theta_{x_0, t}) \,\vert\, x \right] \cdot (\nabla T(x))^T\\
\end{alignat}$$



# Derivation

I'm going to parallel the derivation in [Luhman & Luhman's paper](https://arxiv.org/abs/2207.04316) (appendix A), but with
a different loss function defined for non-gaussian kernels (the denoising-score-matching objective
that [Pascal Vincent showed was asymptotically equivalent to the the explicit score-matching objective](https://www.iro.umontreal.ca/~vincentp/Publications/smdae_techreport.pdf) (section 4.2)).

Vincent derived a score-matching objective with useful properties (changing the variable names a bit):

$$ J_{DSMq}(\theta) = \mathbb{E}_{q(x, z_t)} \left[ \Vert \psi_\theta(z_t; t) - \nabla \ln q(z_t \vert x) \Vert^2 \right] $$

where $\psi_\theta$ is the model, $q(x, z_t)$ is the joint PDF of the data and noised data, and $\nabla$ is with respect to $z_t$.

We can derive a parallel proof to Luhman & Luhman's for this, defining $\bar{x}(z ; t) = \mathbb{E}_{q(x \vert z)}\left[ \nabla \ln q(z_t \vert x_0) \right]$:

$$\begin{alignat}{2}
\argmin_\theta J_{DSMq_t}(\theta) = \, & \argmin_\theta \mathbb{E}_{q(x, z_t)} \left[ \Vert \psi_\theta(z_t; t) - \nabla \ln q_t(z_t \vert x) \Vert^2 \right] \\
    = \, & \argmin_\theta \mathbb{E}_{q(z_t)} \left[ \mathbb{E}_{q(x \vert z_t)} \left[ \Vert \psi_\theta(z_t; t) - \nabla \ln q_t(z_t \vert x) \Vert^2 \right] \right] \\
    = \, & \argmin_\theta \mathbb{E}_{q(z_t)} \left[ \mathbb{E}_{q(x \vert z_t)} \left[ \Vert (\psi_\theta(z_t; t) - \bar{x}(z_t;t)) - (\nabla \ln q_t(z_t \vert x) - \bar{x}(z_t;t)) \Vert^2 \right]  \right] \\
    = \, & \argmin_\theta \mathbb{E}_{q(z_t)} \left[ \mathbb{E}_{q(x \vert z_t)} \left[ \Vert \psi_\theta(z_t; t) - \bar{x}(z_t;t) \Vert^2 - \Vert \nabla \ln q_t(z_t \vert x) - \bar{x}(z_t;t)) \Vert^2 \right]  \right] \\
    = \, & \argmin_\theta \mathbb{E}_{q(z_t)} \left[ \mathbb{E}_{q(x \vert z_t)} \left[ \Vert \psi_\theta(z_t; t) - \bar{x}(z_t;t) \Vert^2 \right]  \right] \\
\end{alignat}$$

This tells us that the optimal score matching model satisfies:

$$ \psi_\theta(z_t; t) = \bar{x}(z ; t) = \mathbb{E}_{q(x \vert z)}\left[ \nabla \ln q(z_t \vert x) \right] $$

If we decide to turn that into a flow model, we should point our differential flow in the direction of $\psi$
to maximally denoise at each step:

$\dot{x} \propto \mathbb{E}_{p(x_0 \vert x; t)}\left[ \nabla \ln p(x \vert x_0; t) \right]$

(This is made more rigorous in connection to the heat equation below)

This can be estimated by training with an L2 loss:

$$ Loss(\theta) = \Vert \psi_\theta(z_t; t) - \nabla \ln p(x \vert x_0; t) \Vert^2 $$

# From the heat equation

With the bayes rule gradient identity from [a few posts ago](https://imh.github.io/2022/11/10/random-handy-probability-identities.html#gradient-of-log-probability-under-bayes-rule)
($\nabla \ln p(x \vert t) = \mathbb{E}_{x_0 \vert x, t} \left[ \nabla \ln \left(p(x \vert x_0, t) \right) \right]$),
this has an alternate form of:

$\dot{x} \propto \nabla \ln p(x \vert t)$

If we add a continuity restriction, ($\partial p/\partial t = -\nabla \cdot p \dot{x}$),
we get:

$$\begin{alignat}{2}
\partial p/\partial t
= & -\nabla \cdot p \dot{x} \\
= & -\nabla p \cdot \dot{x} - p \nabla \cdot \dot{x} \\
\propto & -\nabla p \cdot  \nabla \ln p(x \vert t) - p \nabla \cdot \nabla \ln p(x \vert t) \\
= & -\nabla p \cdot  \frac{\nabla p}{p} - p \nabla \cdot \frac{\nabla p}{p} \\
= & - \frac{1}{p} \nabla p \cdot  \nabla p - p \nabla \cdot \frac{\nabla p}{p} \\
= & - \frac{1}{p} \nabla p \cdot  \nabla p - p (\frac{\nabla \cdot \nabla p}{p} - \frac{\nabla p \cdot \nabla p}{p^2}) \\
= & - \nabla \cdot \nabla p \\
= & - \nabla^2 p \\
\end{alignat}$$

Reversing that, you can derive $\dot{x} \propto \mathbb{E}_{p(x_0 \vert x; t)}\left[ \nabla \ln p(x \vert x_0; t) \right]$ directly from the heat equation.


# The Jacobian

To look at asymptotics, we want the Jacobian of $\dot{x}$.

We'll use [another gradient identity from a few posts ago](https://imh.github.io/2022/11/10/random-handy-probability-identities.html#gradient-of-conditional-expectations)

$$\begin{alignat}{2}

\nabla \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ f(x, x_0) \right]
= & \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla f(x, x_0) + f(x, x_0) \nabla \ln p(x_0 \vert x, t) \right] \\
= & \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla f(x, x_0) + f(x, x_0) \left( \nabla \ln p(x \vert x_0; t) - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] \right) \right] \\

\end{alignat}$$

On with the Jacobian:

$$\begin{alignat}{2}
\nabla \dot{x} \propto & \; \nabla \mathbb{E}_{p(x_0 \vert x; t)}\left[ \nabla \ln p(x \vert x_0; t) \right] \\
=& \; \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ H \ln p(x \vert x_0; t) + \nabla \ln p(x \vert x_0; t) \left( \nabla \ln p(x \vert x_0; t) - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] \right)^T \right] \\
=& \; \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ H \ln p(x \vert x_0; t) \right]
+ \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[\nabla \ln p(x \vert x_0; t) \left( \nabla \ln p(x \vert x_0; t) - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] \right)^T \right] \\
=& \; \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ H \ln p(x \vert x_0; t) \right]
+ \mathbb{V}_{x_0 \sim p(x_0 \vert x; t)} \left[\nabla \ln p(x \vert x_0; t)\right] \\
\end{alignat}$$

(TODO write out how this is equivalent to $\frac{H p(x; t)}{p(x; t)} - \bar{x} \bar{x}^T$) 

# Exponential family conditionals

Exponential family conditionals will take the form

$ \ln p(x \vert x_0 ; t) = \ln h(x) + T(x) \cdot \eta(\theta_{x_0, t}) - A(\theta_{x_0, t}) $

We can get gradients and Hessians:

$ \nabla \ln p(x \vert x_0 ; t) = \nabla \ln h(x) + \nabla T(x) \cdot  \eta(\theta_{x_0, t}) $

$ H \ln p(x \vert x_0 ; t) = H \ln h(x) + D_{jk} T_i(x) \cdot \eta_i(\theta_{x_0, t}) $

Plugging those in, we get:

$$\begin{alignat}{2}
\dot{x} & \propto \mathbb{E}_{p(x_0 \vert x; t)}\left[ \nabla \ln h(x) + \nabla T(x) \cdot  \eta(\theta_{x_0, t}) \right] \\
&= \nabla \ln h(x) + \nabla T(x) \cdot \mathbb{E}_{p(x_0 \vert x; t)}\left[ \eta(\theta_{x_0, t}) \right]
\end{alignat}$$

and

$$\begin{alignat}{2}
\nabla \dot{x} \propto & \; \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ H \ln h(x) + D_{jk} T_i(x) \cdot \eta_i(\theta_{x_0, t}) \right] \\ 
 & + \mathbb{V}_{x_0 \sim p(x_0 \vert x; t)} \left[\nabla \ln h(x) + \nabla T(x) \cdot  \eta(\theta_{x_0, t})\right] \\
=& \; H \ln h(x) + D_{jk} T_i(x) \cdot \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[\eta_i(\theta_{x_0, t}) \right] \\ 
 & + \nabla T(x) \cdot \mathbb{V}_{x_0 \sim p(x_0 \vert x; t)} \left[\eta(\theta_{x_0, t}) \right] \cdot (\nabla T(x))^T\\
\end{alignat}$$

Or more tersely

$$\begin{alignat}{2}
\dot{x} & \propto  \nabla \ln h(x) + \nabla T(x) \cdot \mathbb{E}\left[ \eta(\theta_{x_0, t}) \vert x \right]
\end{alignat}$$

and

$$\begin{alignat}{2}
\nabla \dot{x} \propto & \; H \ln h(x) + D_{jk} T_i(x) \cdot \mathbb{E} \left[\eta_i(\theta_{x_0, t}) \,\vert\, x \right] \\ 
 & + \nabla T(x) \cdot \mathbb{V} \left[\eta(\theta_{x_0, t}) \,\vert\, x \right] \cdot (\nabla T(x))^T\\
\end{alignat}$$
