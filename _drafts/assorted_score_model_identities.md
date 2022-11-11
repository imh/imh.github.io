---
layout: post
title: "Assorted score model/diffusion identities"
toc: true
---

Score model for gaussian noise:

Score modeling gets around learning the normalizing constant in $p(x) = f(x)/Z$ by learning the gradient of the log
probability: $\nabla \log p(x) = \nabla \log f(x)$.

In a denoising model, we have $x_0$ and $x_t$, where $x_0$ is the observed data and $x_t$ is determined by a design
choice: $p(x \vert x_0, t) $.

# Gradient of log probability


$$\nabla \ln p(x \vert t) = \mathbb{E}_{x_0 \vert x, t} \left[ \nabla \ln \left(p(x \vert x_0, t) \right) \right]$$

<details>
    <summary>(expand derivation)</summary>

  $$\begin{alignat}{2}
  
  \nabla \log p(x \vert t) &= \frac{1}{p(x \vert t)} \nabla p(x \vert t) \\
  &= \frac{1}{p(x \vert t)} \nabla \int_{x_0} p(x \vert x_0, t) p(x_0) \\
  &\approx \frac{1}{p(x \vert t)}  \int_{x_0} p(x_0) \nabla p(x \vert x_0, t)  \\
  &= \frac{1}{p(x \vert t)}  \int_{x_0} p(x_0) p(x \vert x_0, t) \nabla \ln \left(p(x \vert x_0, t) \right) \\
  &= \frac{1}{p(x \vert t)}  \int_{x_0} p(x_0 | x, t) p(x \vert t) \nabla \ln \left(p(x \vert x_0, t) \right) \\ \\
  &= \int_{x_0} p(x_0 | x, t) \nabla \ln \left(p(x \vert x_0, t) \right) \\ \\
  &= \mathbb{E}_{x_0 \vert x, t} \left[ \nabla \ln \left(p(x \vert x_0, t) \right) \right] \\
  
  \end{alignat}$$

</details>

For exponential family distributions, this is:

$$\nabla \log p(x \vert t) = \mathbb{E}_{x_0 \vert x, t} \left[ \nabla \ln (h)  + \eta^T \nabla T \right]$$


Alternatively, for a latent variable model, instead of $x_t$ and $x_0$, we'd usually denote it $z_i$ and $x$:

$$\nabla_{z_i} \ln p(z_i) = \mathbb{E}_{x \vert z_i} \left[ \nabla \ln \left(p(z_i \vert x) \right) \right]$$


# Gradient of log conditional probability

$$\nabla \ln p(x_0 \vert x; t) =  \nabla \ln p(x \vert x_0; t) - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] $$

<details>
    <summary>(expand derivation)</summary>

First, let's  look at $\nabla \ln p(x_0 \vert x; t)$.

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

# Gradient of conditional expectations

$$\begin{alignat}{2}

\nabla \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ f(x, x_0) \right]
= & \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla f(x, x_0) + f(x, x_0) \nabla \ln p(x_0 \vert x, t) \right] \\
= & \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla f(x, x_0) + f(x, x_0) \left( \nabla \ln p(x \vert x_0; t) - \mathbb{E}_{x_0 \sim p(x_0 \vert x; t)} \left[ \nabla \ln p(x \vert x_0, t) \right] \right) \right] \\

\end{alignat}$$