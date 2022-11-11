---
layout: post
title: "Variational autoencoders, part 3: Diffusion"
toc: true
---

Now we're getting to the good stuff. This post is about diffusion, like DALL-E and co.

So far we have concerned ourselves with relatively discrete models with a small number of latent variables.
For example, we've looked at models with one or two latent variable.

(graph here)

In this post we'll look at a particular class of models with huge numbers of latent variables, even countably infinite
and uncountably infinite numbers of latent variables.

(graph of diffusion over time)

# Diffusion

Diffusion models have a set of latent variables $z_t$ indexed by time. In general, the logic behind them applies to all
sorts of memoryless processes where we can easily compute the conditional probability $q(z_t | z_s, x)$. 

Recall from the last post that we can decompose an ELBO into a loss function
per data point and latent variable:


$$\begin{alignat}{2}
& ELBO_{i,v} \\
&= - \mathbb{E}_{\text{Pa}(x) \sim q(\cdot \vert x_i, c_i)} \left[ p(x_i \vert \text{Pa}(x), c_i) \right] && \text{ , if } v \text{ is } x \\
&= D_{KL} (q(v \vert x_i, c_i) \Vert p(v \vert c_i)) && \text{ , if } v \text{ has no parents} \\
&= \mathbb{E}_{\text{Pa}(v) \sim q(\cdot \vert x_i, c_i)} \left[ D_{KL} (q(v \vert x_i, c_i, \text{Pa}(v)) \Vert p(v \vert c_i, \text{Pa}(v))) \right] && \text{ , otherwise} \\
\end{alignat}$$

with a total ELBO:

$$ELBO = \sum_{\substack{i \in \text{Data} \\ v \in \text{variables}}} ELBO_{i,v}$$
