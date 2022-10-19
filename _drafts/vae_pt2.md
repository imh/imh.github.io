---
layout: post
title: "Variational autoencoders, part 2: Adding structure inside the latent model"
toc: true
---

# Recap
In the last post, we showed that we can model an unknown data distribution $q(x)$ with a fitted model $p(x)$ by minimizing
the KL divergence between them.

We also showed that simple models of $p(x)$ can fall flat, but we can add latent variables $z$ to make simple models
more effective, building up to the evidence lower bound (ELBO) to show that fitting a model to the latent structure
bounds the error on the original data.

KL(x, z) >= KL(x)

## This time

This time, we'll look at how to add further structure to the latent model $p(z)$.

z is a bunch of variables:

$$ z = a, b, c $$

We can model x and z as a big messy joint distribution $p(x, z) = p(x,a,b,c)$, or we can factor it as a bunch of conditional 
probability distributions. For example:

$$ p(x, z) = p(x, a,b,c) = p(x|a, b) p(a | c) p(b) p(c) $$

This forms a graph, with each variable conditioned on its parents:

(graph here)

# Toy example

We can extend the previous toy example to form a deeper hierarchy (you might be starting to notice
that I'm building towards super-resolution models in later posts):

(two-level clustering graph)

$p(x, y, z) = p(x|y)p(y|z)p(z)$

# more generally

Denote the set of variables as $V = {x, a,b,c}$ and the set of variables in z as $Z = {a,b,c}$. 

Denote $\text{Pa(v)}$ as the set of parents of $v$ in the graph. For example, $\text{Pa(x)} = \{a,b\}$.

We can now write the joint distribution as:

$$ p(x, z) = \prod_{v \in V} p(v \vert \text{Pa}(v)) $$

# The evidence lower bound

This lets us write a factored version of the ELBO

Recall my favorite form of the ELBO from the last post:

$$\begin{multline}
ELBO =  \mathbb{E}_{x \sim q} \left[ D_{KL} ( q(z|x) \Vert p(z)) \right] \\
 - \mathbb{E}_q \left[ \ln p(x|z) \right] 
\end{multline}$$

Rewriting the first term:

$$\begin{align}
& \mathbb{E}_{x \sim q} \left[ D_{KL} ( q(z|x) \Vert p(z)) \right] \\
&= \mathbb{E}_{x \sim q} \left[ \mathbb{E}_{z \sim q(z|x)} \left[ \ln \frac{q(z|x)}{p(z)}\right]\right] \\
&= \mathbb{E}_{x \sim q} \left[ \mathbb{E}_{z \sim q(z|x)} \left[ \ln \frac{\prod_{v \in Z} q(v \vert \text{Pa}(v), x)}{\prod_{v \in Z} p(v \vert \text{Pa}(v))}\right]\right] \\
&= \mathbb{E}_{x \sim q} \left[ \mathbb{E}_{z \sim q(z|x)} \left[\sum_{v \in Z} \ln \frac{q(v \vert \text{Pa}(v), x)}{p(v \vert \text{Pa}(v))}\right]\right] \\
&= \sum_{v \in Z} \mathbb{E}_{x \sim q} \left[ \mathbb{E}_{z \sim q(z|x)} \left[ \ln \frac{q(v \vert \text{Pa}(v), x)}{p(v \vert \text{Pa}(v))}\right]\right] \\
&= \sum_{v \in Z} \mathbb{E}_{x \sim q} \left[ \mathbb{E}_{v, \text{Pa}(v) \sim q(v, \text{Pa}(v)|x)} \left[ \ln \frac{q(v \vert \text{Pa}(v), x)}{p(v \vert \text{Pa}(v))}\right]\right] \\
&= \sum_{v \in Z} \mathbb{E}_{x \sim q} \left[ \mathbb{E}_{\text{Pa}(v) \sim q(\text{Pa}(v)|x)} \left[ D_{KL}(q(v \vert \text{Pa}(v), x) \Vert p(v \vert \text{Pa}(v))) \right]\right] \\
&= \sum_{v \in Z} \mathbb{E}_{x, \text{Pa}(v) \sim q} \left[ D_{KL}(q(v \vert \text{Pa}(v), x) \Vert p(v \vert \text{Pa}(v))) \right] \\
\end{align}$$

We see that it's the sum of KL divergences of each part of the graph covering the latent variables, but also conditioned on
$x$.

We can rewrite the second term of the ELBO as well:

$$\begin{align}
& \mathbb{E}_q \left[ \ln p(x|z) \right] \\
&= \mathbb{E}_q \left[ \ln p(x|\text{Pa}(x)) \right] \\
&= \mathbb{E}_{x, \text{Pa}(x) \sim q} \left[ \ln p(x|\text{Pa}(x)) \right]
\end{align}$$

This gives us a factored form of the ELBO:

$$\begin{multline}
ELBO =  \\
\sum_{v \in Z} \mathbb{E}_{x, \text{Pa}(v) \sim q} \left[ D_{KL}(q(v \vert \text{Pa}(v), x) \Vert p(v \vert \text{Pa}(v))) \right] \\
 - \mathbb{E}_{x, \text{Pa}(x) \sim q} \left[ \ln p(x|\text{Pa}(x)) \right] 
\end{multline}$$

**This tells us that we can decompose our model into a loss function into a bunch of smaller loss functions for each
part of our model.**

## Back to the toy model

Applying this form of the ELBO to the toy model, we get
 
$$\begin{alignat}{3}
& ELBO &&  \\
&= && \sum_{v \in Z} \mathbb{E}_{x, \text{Pa}(v) \sim q} \left[ D_{KL}(q(v \vert \text{Pa}(v), x) \Vert p(v \vert \text{Pa}(v))) \right] \\
& && - \mathbb{E}_{x, \text{Pa}(x) \sim q} \left[ \ln p(x|\text{Pa}(x)) \right] \\
&= && \mathbb{E}_{x, y \sim q} \left[ D_{KL}(q(y \vert z, x) \Vert p(y \vert z)) \right] \\
& && + \mathbb{E}_{x, z \sim q} \left[ D_{KL}(q(z \vert x) \Vert p(z)) \right] \\
& && - \mathbb{E}_{x, y \sim q} \left[ \ln p(x|y) \right]
\end{alignat}$$

Like last time, we can keep things simple. We will fit:
- logistic models for $q(y \vert x)$ and $q(z \vert y)$
- gaussian models for $p(x \vert y)$ and $p(y \vert z)$
- a uniform "model" for $p(z)$

Note that we can compute $q(y \vert z, x)$ with bayes rule (todo details in demo).

(gif showing the model fit and sampling)

We can then sample by sampling $z \sim p(z)$, then $y \sim p(y \vert z)$, then $x \sim p(x \vert y)$.

(gif showing sampling)


# Why is this useful?

This form is really handy for complicated models, as we'll see in the next post covering diffusion.

In general, this form is useful when:
- We can jointly sample $ (x, \text{Pa}(v)) \sim q $
- We can compute $p(v \vert \text{Pa}(v))$
- We can compute $q(v \vert \text{Pa}(v), x)$ (use bayes rule!)

These criteria guide our choices for variational models.

A crucial point here is that the graph structure (which variables are parents of which) is up to us.
**It doesn't have to correspond to the actual structure of q or p.**
You only need the criteria listed above for it to be useful.