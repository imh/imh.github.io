---
layout: post
title: "[WIP] Canonical probability flow ODE"
toc: true
---

\[**Under construction. If you see this it's probably because I shared it directly with you.**\]

# tl;dr

Karras et al have a nicer version of Song's probability flow ODE, but it can be made even clearer

(to do: figure out how
jekyll and bibtex work together).

Specifically, it reduces to (without loss of generality, up to a straightforward reparametrization):

$$\dot{x} = x - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ x_0 \right] =: x - m(x; t)$$

where $m(x; t)$ is the average initial value of the diffusion process given $x_t$.

It has a really nice Jacobian too:

$ \nabla \dot{x} = I  - \frac{1}{\sigma^2} \mathbb{V}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nu \right] $

where $\nu(x_0, x, t) = x_0 - m(x; t)$.

Higher order derivatives are also super simple (see below.)

Lastly, this makes it really clear what behavior is as $t \to 0$ and $t \to \infty$, because we can look at what happens
to $m$ as $t \to 0$ and $t \to \infty$. (e.g. $m \to \mathbb{E}[x_0]$ as $t \to \infty$.)

# Derivation


It's useful because sets of trajectories that follow this ODE preserve marginal distributions. That is, if you sample
$x \sim p(x_0)$ and integrate either the ODE or the SDE, you'll end up with $p(x_t)$ the same either way. You can also
time-reverse it and go from random noise to the data distribution deterministically (but sampling is more stable; see
langevin dynamics).

In Song's paper, they frame it as:

$$ d x = f d t + g d W $$

Leading to an ODE of:

$$ \dot{x} = f - \frac{1}{2} g^2 \nabla \ln p $$

In Karras's paper, they reparametrize from $ (f, g)$ to $(s, \sigma)$, giving:

$$ \dot{x} = \frac{\dot{s}}{s} x - s^2 \dot{\sigma} \sigma \nabla \ln p $$

There are much simpler ways to look at it. First, we simplify $\nabla \ln p$.

$$ \nabla \ln p(x; t) = \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] $$

<details>
    <summary>(expand derivation)</summary>

$$\begin{alignat}{2}
\nabla \ln p(x; t) = & \frac{1}{p(x; t)} \nabla p(x; t) \\
= & \frac{1}{p(x; t)} \nabla \int p(x \vert x_0, t) p(x_0) d x_0 \\
= & \frac{1}{p(x; t)}\int \nabla p(x \vert x_0, t) p(x_0) d x_0 \\
= & \frac{1}{p(x; t)}\int p(x \vert x_0, t) p(x_0) \nabla \ln p(x \vert x_0, t) d x_0 \\
= & \frac{1}{p(x; t)}\int p(x_0 \vert x; t) p(x; t) \nabla \ln p(x \vert x_0, t) d x_0 \\
= & \int p(x_0 \vert x; t) \nabla \ln p(x \vert x_0, t) d x_0 \\
= & \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] \\
\end{alignat}$$

</details>

The ODEs become

$$ \dot{x} = f - \frac{1}{2} g^2 \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] $$

and

$$ \dot{x} = \frac{\dot{s}}{s} x - s^2 \dot{\sigma} \sigma \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] $$

This still isn't restricted to gaussians, but with gaussian $p(x \vert x_0; t)$, it gets even simpler.

Karras's form is especially easy because $$ p(x \vert x_0, t) $$ is directly expressed in terms of $s$ and $\sigma$:
$$ p(x \vert x_0, t) = N_x (s x_0, s^2 \sigma^2 I) $$.

So the log gradient is
$  \nabla \ln p(x \vert x_0, t) = \frac{s x_0 - x}{s^2 \sigma^2} $.

Karras's ODE becomes:

$$
\begin{alignat}{2}
\dot{x}
\dot{x} = & \frac{\dot{s}}{s} x - s^2 \dot{\sigma} \sigma \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \frac{s x_0 - x}{s^2 \sigma^2} \right] \\
\dot{x} = & \frac{\dot{s}}{s} x  + s^2 \dot{\sigma} \sigma \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \frac{x}{s^2 \sigma^2} \right] - s^2 \dot{\sigma} \sigma \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \frac{s x_0 }{s^2 \sigma^2} \right] \\
\dot{x} = & \frac{\dot{s}}{s} x + s^2 \dot{\sigma} \sigma  \frac{x}{s^2 \sigma^2} - s^2 \dot{\sigma} \sigma \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \frac{s x_0 }{s^2 \sigma^2} \right]  \\
\dot{x} = & \frac{\dot{s}}{s} x + \frac{\dot{s \sigma}}{\sigma} \frac{x}{s} - \frac{s \dot{\sigma}}{\sigma} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \frac{x_0 }{s} \right]  \\
\frac{\dot{x}}{s} = & \frac{\dot{s}}{s} \frac{x}{s} + \frac{\dot{\sigma}}{\sigma} \frac{x}{s} - \frac{\dot{\sigma}}{\sigma} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \frac{x_0 }{s} \right]  \\

\end{alignat}$$

Noting that $\frac{d}{dt} (x / s) = \dot{x} / s - x \dot{s} / s^2$, we get:

$$
\begin{alignat}{2}
\frac{d}{dt} \left(\frac{x}{s} \right) = &  \frac{\dot{\sigma}}{\sigma} \frac{x}{s} - \frac{\dot{\sigma}}{\sigma} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \frac{x_0 }{s} \right]  \\

\end{alignat}$$

Since the transformation from $x \rightarrow x/s$ is invertible, we can ignore the scale factor without loss of generality (imagine that we'd just changed variables to $x' = x/s$ or something).

$$
\begin{alignat}{2}
\dot{x} = & \frac{\dot{\sigma}}{\sigma} x - \frac{\dot{\sigma}}{\sigma} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ x_0 \right]   \\
= & \frac{\dot{\sigma}}{\sigma} \left( x - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ x_0 \right]  \right) \\
= & \left(\frac{d}{d t} \ln \sigma \right) \left( x - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ x_0 \right]  \right) \\
\end{alignat}$$

We can define a new time term $t'$ such that $dt'/dt = \frac{d}{d t} \ln \sigma$ and get what I'd
consider a pretty fundamental form of the diffusion ODE.

# The "canonical" form

$$\dot{x} = x - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ x_0 \right] =: x - m(x; t)$$

It has to be a standard result, but I haven't seen it before.

At any given point in time, the paths must be directly towards or away from the (average) place they started.
You can change time around to rescale things however, but these tangent vectors are independent of parametrization.

# Derivatives and stability

It turns out that $m(x; t)$ has really straightforward derivatives, letting us look at the ODE's properties fairly easily.

## Derivatives

First, let's invert $\nabla \ln p(x_0 \vert x; t)$ with Bayes rule.

$$\nabla \ln p(x_0 \vert x; t) =  \nabla \ln p(x \vert x_0; t) - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] $$

<details>
    <summary>(expand derivation)</summary>

$$\begin{alignat}{2}

\nabla \ln p(x_0 \vert x; t) = & \nabla \ln \frac{p(x \vert x_0; t) p(x_0)}{p(x; t)} \\
 = & \nabla \ln p(x \vert x_0; t) - \nabla \ln p(x; t) \\
 = & \nabla \ln p(x \vert x_0; t) - \frac{1}{p(x; t)} \nabla p(x; t) \\
 = & \nabla \ln p(x \vert x_0; t) - \frac{1}{p(x; t)} \nabla p(x; t) \\
 = & \nabla \ln p(x \vert x_0; t) - \frac{1}{p(x; t)} \nabla \int p(x \vert x_0, t) p(x_0) d x_0 \\
 = & \nabla \ln p(x \vert x_0; t) - \frac{1}{p(x; t)} \int \nabla p(x \vert x_0, t) p(x_0) d x_0 \\
 = & \nabla \ln p(x \vert x_0; t) - \frac{1}{p(x; t)} \int p(x \vert x_0, t) p(x_0) \nabla \ln p(x \vert x_0, t) d x_0 \\
 = & \nabla \ln p(x \vert x_0; t) - \frac{1}{p(x; t)} \int p(x_0 \vert x, t) p(x; t) \nabla \ln p(x \vert x_0, t) d x_0 \\
 = & \nabla \ln p(x \vert x_0; t) - \frac{1}{p(x; t)} \int p(x_0 \vert x, t) p(x; t) \nabla \ln p(x \vert x_0, t) d x_0 \\
 = & \nabla \ln p(x \vert x_0; t) - \int p(x_0 \vert x, t) \nabla \ln p(x \vert x_0, t) d x_0 \\
 = & \nabla \ln p(x \vert x_0; t) - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] \\

\end{alignat}$$

</details>

This is handy in general, but for Karra's model, we specifically had $\nabla \ln p(x \vert x_0, t) = \frac{s x_0 - x}{s^2 \sigma^2}$.
We already showed that we can set $s=1$ without loss of generality, so $\nabla \ln p(x \vert x_0, t) = \frac{x_0 - x}{\sigma^2}$

$$\begin{alignat}{2}

\nabla \ln p(x_0 \vert x; t)
=& \nabla \ln p(x \vert x_0; t) - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] \\
= & \frac{x_0 - x}{\sigma^2} - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \frac{x_0 - x}{\sigma^2} \right] \\
= & \frac{x_0}{\sigma^2} - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \frac{x_0 }{\sigma^2} \right] \\
= & \frac{x_0-m}{\sigma^2} \\

\end{alignat}$$

Now we can also easily differentiate conditional expectations:

$$\begin{alignat}{2}

\nabla \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ f(x, x_0) \right] 
= & \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla f(x, x_0) + f(x, x_0) \nabla \ln p(x_0 \vert x, t) \right] \\
= & \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla f(x, x_0) + f(x, x_0) \left( \frac{x_0 - m}{\sigma^2} \right) \right] \\

\end{alignat}$$


(The $x_0 - m$ term might be transposed, depending on which axis we want to apply the gradient with respect to.)

Let's use that to differentiate $m(x; t)$.

$$\begin{alignat}{2}

\nabla m(x; t)
= & \nabla \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ x_0 \right] \\
= & \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ x_0 \left( \frac{x_0^T - m^T}{\sigma^2} \right) \right] \\
= & \frac{1}{\sigma^2} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ x_0 \left( x_0^T - m^T \right) \right] \\
= & \frac{1}{\sigma^2} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ x_0 \left( x_0 - m \right)^T \right] \\
= & \frac{1}{\sigma^2} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ (x_0 - m) \left( x_0 - m \right)^T \right] \\
= & \frac{1}{\sigma^2} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nu \nu^T \right] \\
= & \frac{1}{\sigma^2} \mathbb{V}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nu  \right] \\

\end{alignat}$$

where we've defined $\nu(x, x_0; t) = x_0 - m(x_0; t)$. 

We can use the $ \nabla \ln p(x_0 \vert x; t) = \frac{x_0-m}{\sigma^2} = \frac{\nu}{\sigma^2}$  identity to take take the hessian of $m(x; t)$:

$$\begin{alignat}{2}

D_{jk} m_i(x; t)
= & \frac{1}{\sigma^2} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nu_i \nu_j \frac{\nu_k}{\sigma^2} \right] \\
= & \frac{1}{\sigma^4} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nu_i \nu_j \nu_k \right] \\

\end{alignat}$$



Or even more generally for the $n$-th derivative:

$$\begin{alignat}{2}

D_{j_1...j_n} m_i(x; t)
= & \frac{1}{\sigma^{2 n}} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nu_{j_1} \nu_{j_2} ...  \nu_{j_n}  \nu_i \right] \\
\end{alignat}$$

The $n$-th derivative of $m$ is proportional to $(n+1)$-th moment of $\nu$ (comparable
to the moment generating function of $\nu$).

We can contract that in a particular direction to get the taylor expansion of $m(x; t)$ (repeated indices are contracted over):

$$\begin{alignat}{2}

m_i(x; t)
= & \sum_n \frac{1}{\sigma^{2 n}} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nu_{j_1} \nu_{j_2} ...  \nu_{j_n}  \nu_i \right] \Delta x_{j_1} \Delta x_{j_2} ... \Delta x_{j_n} \\
= & \sum_n \frac{1}{\sigma^{2 n}} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ (\nu_{j_1} \Delta x_{j_1}) (\nu_{j_2} \Delta x_{j_2}) ...  (\nu_{j_n} \Delta x_{j_n}) \nu_i \right] \\
= & \sum_n \frac{1}{\sigma^{2 n}} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ (\nu \cdot \Delta x)^n \nu_i \right] \\
= & \sum_n \frac{1}{\sigma^{2 n}} \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \Vert \nu \Vert^n \Vert \Delta x \Vert^n \cos^n \left(\theta_{\nu, \Delta x} \right) \nu_i \right] \\
\end{alignat}$$

Going back to single-order derivatives, we linearize the ODE:

$$\begin{alignat}{2}
 \dot{x} = & \, x - m \\
 \approx & \, x - m(x'; t) - \frac{1}{\sigma^2} \mathbb{E}_{x_0 \sim p(x_0 \vert x'; t)} \left[\nu \nu^T  \right] \Delta x \\
  = & \, x - m(x'; t) - \frac{1}{\sigma^2} \mathbb{V}_{x_0 \sim p(x_0 \vert x'; t)} \left[ \nu \right] \Delta x
\end{alignat}$$

Finally, we can look at stability properties, which are interesting for this problem.

The jacobian is straightforward as $ I  - \frac{1}{\sigma^2} \mathbb{V}_{x_0 \sim p(x_0 \vert x'; t)} \left[ \nu \right] $.

To look at stability, we might as well scale it by $-\sigma^2$ (time reversal to reverse diffusion) to get $ \mathbb{V}_{x_0 \sim p(x_0 \vert x'; t)} \left[ \nu \right] - \sigma^2 I $.


## Stability

The eigendecomposition tells us a few things, if I remember correctly:

- Which axes are stable/unstable?
- Where are the stable manifolds?
- Where might there be bifurcations? This branching during the reverse diffusion step is what lets us get a multi-modal distribution out. How many modes/when/where?
- How do bifurcations appear/disappear as a function of $\sigma$? It would tell us a minimum noise, relative to the data scale to prevent overfitting. It would also tell us the complexity of tree that could approximate the distribution (as a mixture of how many gaussians)
- What are the relevant time-scales? How fast does it converge? If we have a degenerate data distribution, how fast does it converge to the manifold? How fast does it converge within the manifold?

To that end, I should take a look at some toy distributions and see if I can calculate $m$. In particular, I'm
interested in comparing non-degenerate data distributions with degenerate data distributions that have the same moments.
So a bunch of distributions with overall zero mean and identity covariance.

Maybe some nice toy models would be

- A multivariate gaussian.

- The uniform distribution on a ellipsoidal shell

- points around the shell of an ellipsoid

- - Two degenerate gaussians parallel with each other

- Two degenerate gaussians that intersect

## Limits

to-do: limits for those toy models

# Extensions


Other interesting things to look at would be how this generalizes to non-gaussian noise. Are there useful analogies
in the whole exponential family? Maybe following Karras's idea of using the marginals as first class citizens could extend
to exponential family marginals?  $\nabla ln p$ is relatively straightforward for them. What's the diffusion process
to make an ODE out of? If Dirichlet has a useful  analog, then this could noise and denoise categories in a way
that handles branching through bifurcations rather than autoregressive sampling like typical language models.


