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