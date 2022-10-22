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

$$
D_{KL}(q(x, z) \Vert p(x, z))
\ge D_{KL}( q(x) \Vert p(x) )
$$

Rewriting this inequality, we can see that we minimize $D_{KL}( q(x) \Vert p(x) )$ by minimizing the ELBO:

$$\begin{alignat}{2}
ELBO_i = & D_{KL} ( q(z \vert x_i) \Vert p(z)) \\ 
         & - \mathbb{E}_{z \sim q(z \vert x_i)} \left[ \ln p(x_i \vert z) \right] \\
ELBO =   & \mathbb{E}_i \left[ ELBO_i \right]
\end{alignat}$$

## This time

We'll add structure on $x$ and $z$ and get a new ELBO.

$$x' = (x, c)$$

$$z' = (z_1, z_2, ...)$$

We'll say that $x$ is the part of $x'$ that we model as downstream of $z$ and $c$ is the part of $x'$ that we model
as upstream of $z$.

We can write a new ELBO as

$$\begin{alignat}{2}
& D_{KL}(q(x', z') \Vert p(x', z')) \\
&= D_{KL}(q(x, c, z) \Vert p(x, c, z)) \\
&= \mathbb{E}_{(x, c, z) \sim q} \left[ \ln \frac{q(x, c, z)}{p(x, c, z)} \right] \\
&= \mathbb{E}_{(x, c, z) \sim q} \left[ \ln \frac{q(z \vert x, c) q(x, c)}{p(x, z \vert c) p(c)} \right] \\
&= \mathbb{E}_{(x, c, z) \sim q} \left[ \ln \frac{q(z \vert x, c) q(x, c)}{p(x \vert z, c) p(z \vert c) p(c)} \right] \\
&= \mathbb{E}_{(x, c) \sim q} \left[ D_{KL} (q(z \vert x, c) \Vert p(z \vert c)) \right] - \mathbb{E}_{(x, c, z) \sim q} \left[ p(x \vert z, c) \right] - \mathbb{E}_{c \sim q} \left[ p(c) \right]- H(q(x, c)) \\
\end{alignat}$$

$H(q(x, c))$ is fixed by the data. This gives us a conditional ELBO:

$$
ELBO_i = D_{KL} (q(z \vert x_i, c_i) \Vert p(z \vert c_i))  - \mathbb{E}_{z \sim q} \left[ p(x_i \vert z, c_i) \right] - \mathbb{E}_{c \sim q} \left[ p(c) \right]
$$

Further, if $q(z \vert x, c)$ and $p(z \vert c)$ decompose with the same factorization (I think this requires that
their moralizations/markov equivalents are the same. gotta double check the PGM book), then we can decompose the KL term
in a straightforward way.

That is, we're requiring a graphical structure such that, given the same mapping $\text{Pa}$ from variables to parents:

$$q(z \vert x, c) = \prod_{j \in Z} q(v_j \vert x, c, \text{Pa}(v_j))$$

$$p(z \vert c) = \prod_{j \in Z} p(v_j \vert c, \text{Pa}(v_j))$$

Then the KL terms decomposes as:

$$\begin{alignat}{2}
& D_{KL} (q(z \vert x_i, c_i) \Vert p(z \vert c_i)) \\
&= \mathbb{E}_{z \sim q(\cdot \vert x_i, c_i)} \left[ \ln \frac{\prod_{j \in Z} q(v_j \vert x, c, \text{Pa}(v_j))}{\prod_{j \in Z} p(v_j \vert c, \text{Pa}(v_j))} \right] \\
&= \sum_{j \in Z} \mathbb{E}_{z \sim q(\cdot \vert x_i, c_i)} \left[ \ln \frac{q(v_j \vert x_i, c_i, \text{Pa}(v_j))}{p(v_j \vert c_i, \text{Pa}(v_j))} \right] \\
&= \sum_{j \in Z} \mathbb{E}_{z \sim q(\cdot \vert x_i, c_i)} \left[ \ln \frac{q(v_j \vert x_i, c_i, \text{Pa}(v_j))}{p(v_j \vert c_i, \text{Pa}(v_j))} \right] \\
&= \sum_{j \in Z} \mathbb{E}_{\text{Pa}(v_j) \sim q(\cdot \vert x_i, c_i)} \left[ D_{KL} (q(v_j \vert x_i, c_i, \text{Pa}(v_j)) \Vert p(v_j \vert c_i, \text{Pa}(v_j))) \right]
\end{alignat}$$

If a variable $v_j$ has no parents, then $\text{Pa}(v_j) = \emptyset$ and the KL term is just the KL term for that variable.

We can now write the ELBO per variable, as:

$$\begin{alignat}{2}
& ELBO_{i,j} \\
&= - \mathbb{E}_{z \sim q(\cdot \vert x_i, c_i)} \left[ p(x_i \vert z, c_i) \right] && \text{ , if } v_j \text{ is } x \\
&= - \mathbb{E}_{c \sim q} \left[ p(c_i) \right] && \text{ , if } v_j \text{ is } c \\
&= D_{KL} (q(v_j \vert x_i, c_i) \Vert p(v_j \vert c_i)) && \text{ , if } v_j \text{ has no parents} \\
&= \mathbb{E}_{\text{Pa}(v_j) \sim q(\cdot \vert x_i, c_i)} \left[ D_{KL} (q(v_j \vert x_i, c_i, \text{Pa}(v_j)) \Vert p(v_j \vert c_i, \text{Pa}(v_j))) \right] && \text{ , otherwise} \\
\end{alignat}$$

Note that by "$v_j$ has no parents" we specifically mean that $q(v_j \vert x_i, c_i, \text{Pa}(v_j)) = q(v_j \vert x_i, c_i)$
and $p(v_j \vert c_i, \text{Pa}(v_j)) = p(v_j \vert c_i)$, so it means that it has no parents in $q$,
conditional on $x$ and $c$, and no parents in $p$, conditional on $c$, which is less restricting than just
"has no parents at all."

This gives us a total ELBO:

$$ELBO = \sum_{i \in \text{Data}} \sum_{j \in \text{variables}} ELBO_{i,j}$$

For observed data $v = x$ and $v = c$, the ELBO is a cross entropy.

For generated latent data $v \in z$, the ELBO is a KL divergence.

This splits the optimization problem into separate parts per variable, which is very convenient.
It also allows us to limit any sampling to one variable at a time.

## Back to the toy model

Applying this form of the ELBO to the toy model and using the factorization we defined for $p$, we get

$$\begin{alignat}{2}
ELBO_{i,x} &= - \mathbb{E}_{z_1 \sim q(z_1 \vert x_i)} \left[ \ln p(x_i \vert z_1) \right] \\
ELBO_{i,z_1} & = \mathbb{E}_{z_2 \sim q(z_2 \vert x_i)} \left[ D_{KL}(q(z_1 \vert z_2, x_i) \Vert p(z_1 \vert z_2)) \right] \\
ELBO_{i,z_2} &= D_{KL}(q(z_2 \vert x_i) \Vert p(z_2)) \\
\end{alignat}$$

 
$$\begin{alignat}{3}
& && ELBO_i   \\
&= && ELBO_{i,x} + ELBO_{i,z_1} + ELBO_{i,z_2} \\
&= && - \mathbb{E}_{z_1 \sim q(z_1 \vert x_i)} \left[ \ln p(x_i \vert z_1) \right] \\
& && + \mathbb{E}_{z_2 \sim q(z_2 \vert x_i)} \left[ D_{KL}(q(z_1 \vert z_2, x_i) \Vert p(z_1 \vert z_2)) \right] \\
& && + D_{KL}(q(z_2 \vert x_i) \Vert p(z_2)) \\
\end{alignat}$$

and finally

$$ELBO = \sum_{i \in \text{Data}} ELBO_i$$

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