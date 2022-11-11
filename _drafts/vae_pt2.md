---
layout: post
title: "Variational autoencoders, part 2: Adding structure inside the latent model"
toc: true
---

# Recap
In the last post, we showed that we can model an unknown data distribution $q(x)$ with a fitted model $p(x)$ by minimizing
the KL divergence between them.

We also showed that simple models of $p(x)$ can fall flat, but we can add latent variables $z$ to make simple models
more effective, building up to an evidence lower bound (ELBO) to show that fitting a model to the latent structure
bounds the error on the original data.

$$
D_{KL}(q(x, z) \Vert p(x, z))
\ge D_{KL}( q(x) \Vert p(x) )
$$

Rewriting this inequality, we can see that we minimize $D_{KL}( q(x) \Vert p(x) )$ by minimizing a per-datapoint ELBO:

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

(Note: $c$ can be be latent or observed, but we're structuring things so that it's not learned,
meaning if it's latent, we are choosing how to sample it rather than learning how to sample it.
It's anything we want to condition on. So it could just as well be:)

$$x' = (x)$$

$$z' = (c, z_1, z_2, ...)$$


## Toy model

Suppose our data $(x, c)$looks like this:

![Distribution x ~ q]({{ site.url }}/assets/vae/pt2/qx.jpeg)

As before, directly $q(x)$ as a gaussian is a bad idea:

![Distribution x ~ q]({{ site.url }}/assets/vae/pt2/qx_learned_px.jpeg)

But this time, modeling $q(x)$ like we did last time leaves lots of unmodeled structure:

![Distribution x ~ q, p(x) with simple latent]({{ site.url }}/assets/vae/pt2/qx_learned_px_one_level.jpeg)

This time, we'll use two latent variables $z_1$ and $z_2$ to model the hierarchy:

![Distribution (x, z, c) ~ q]({{ site.url }}/assets/vae/pt2/qxcz.jpeg)

This kind of hierarchy might show up in a superresolution task, where we have successive
one-to-many relationships. In that case, we might say that:

- $x$ is the highest-resolution image
- $z_1$ is medium-resolution images
- $z_2$ is low-resolution images
- $c$ is the class label

Or we could even do it with text:

- $x$ is the words
- $z_1$ is sentences encodings
- $z_2$ is document encodings
- $c$ is pre-determined a sampling scheme

# Formalizing it

We can write a new ELBO as

$$\begin{alignat}{2}
& D_{KL}(q(x', z') \Vert p(x', z')) \\
&= D_{KL}(q(x, c, z) \Vert p(x, c, z)) \\
&= \mathbb{E}_{(x, c, z) \sim q} \left[ \ln \frac{q(x, c, z)}{p(x, c, z)} \right] \\
&= \mathbb{E}_{(x, c, z) \sim q} \left[ \ln \frac{q(z \vert x, c) q(x, c)}{p(x, z \vert c) p(c)} \right] \\
&= \mathbb{E}_{(x, c, z) \sim q} \left[ \ln \frac{q(z \vert x, c) q(x, c)}{p(x \vert z, c) p(z \vert c) p(c)} \right] \\
&= \mathbb{E}_{(x, c) \sim q} \left[ D_{KL} (q(z \vert x, c) \Vert p(z \vert c)) \right] - \mathbb{E}_{(x, c, z) \sim q} \left[ p(x \vert z, c) \right] - \mathbb{E}_{c \sim q} \left[ p(c) \right]- H(q(x, c)) \\
\end{alignat}$$

The requirements mentioned above about $c$ mean that $H(q(x, c))$ and $\mathbb{E}_{c \sim q} \left[ p(c) \right]$ are
fixed, and so can be ignored. This gives us a conditional ELBO:

$$
ELBO_i = D_{KL} (q(z \vert x_i, c_i) \Vert p(z \vert c_i))  - \mathbb{E}_{z \sim q} \left[ p(x_i \vert z, c_i) \right]
$$

We also want to split the variables in $z$, but to stay general, we won't split into individual variables.
Instead we'll partition the set of variables $Z = \\{z_1, z_2, ...\\}$ in sets $V_Z = \\{v_1, v_2, ...\\}$ such that
V forms a partitioning of $Z$ (that is, $Z = \bigcup_{v \in V_Z} v$ and $v_i \cap v_j = \emptyset$ for $i \ne j$).

We'll choose this partitioning such that the following are both valid (conditional) factorizations of $q$ and $p$:

$$q(z \vert x, c) = \prod_{v \in V_Z} q(v \vert x, c, \text{Pa}(v))$$

$$p(z \vert c) = \prod_{v \in V_Z} p(v \vert c, \text{Pa}(v))$$

where $\text{Pa}(v)$ is the set of parents of $v$ in the graph defined by $p(z \vert c)$,
meaning we can sample $z$ by sampling each of its parents.

To find the right partitioning of $Z$, realize that we only need to group where we are flipping parent/child
relationships in $q$. The nodes that are grouped are precisely the ones grouped by moralizing a bayes network into a
markov network. Conditioning on a child groups the parents, so if $z_i \rightarrow z_j \leftarrow z_k$ in $q$, then
$z_j \rightarrow \\{ z_i, z_k \\}$ in $p$, to get a clean partitioning.

Then the KL terms decomposes as:

$$\begin{alignat}{2}
& D_{KL} (q(z \vert x_i, c_i) \Vert p(z \vert c_i)) \\
&= \mathbb{E}_{z \sim q(\cdot \vert x_i, c_i)} \left[ \ln \frac{\prod_{v \in V_Z} q(v \vert x, c, \text{Pa}(v))}{\prod_{v \in V_Z} p(v \vert c, \text{Pa}(v))} \right] \\
&= \sum_{v \in V_Z} \mathbb{E}_{z \sim q(\cdot \vert x_i, c_i)} \left[ \ln \frac{q(v \vert x_i, c_i, \text{Pa}(v))}{p(v \vert c_i, \text{Pa}(v))} \right] \\
&= \sum_{v \in V_Z} \mathbb{E}_{z \sim q(\cdot \vert x_i, c_i)} \left[ \ln \frac{q(v \vert x_i, c_i, \text{Pa}(v))}{p(v \vert c_i, \text{Pa}(v))} \right] \\
&= \sum_{v \in V_Z} \mathbb{E}_{\text{Pa}(v) \sim q(\cdot \vert x_i, c_i)} \left[ D_{KL} (q(v \vert x_i, c_i, \text{Pa}(v)) \Vert p(v \vert c_i, \text{Pa}(v))) \right]
\end{alignat}$$

If a variable $v$ has no parents, then $\text{Pa}(v) = \emptyset$ and the KL term is just the KL term for that variable.

We can now write the ELBO per variable, as:

$$\begin{alignat}{2}
& ELBO_{i,v} \\
&= - \mathbb{E}_{\text{Pa}(x) \sim q(\cdot \vert x_i, c_i)} \left[ p(x_i \vert \text{Pa}(x), c_i) \right] && \text{ , if } v \text{ is } x \\
&= D_{KL} (q(v \vert x_i, c_i) \Vert p(v \vert c_i)) && \text{ , if } v \text{ has no parents} \\
&= \mathbb{E}_{\text{Pa}(v) \sim q(\cdot \vert x_i, c_i)} \left[ D_{KL} (q(v \vert x_i, c_i, \text{Pa}(v)) \Vert p(v \vert c_i, \text{Pa}(v))) \right] && \text{ , otherwise} \\
\end{alignat}$$

Note that by "$v$ has no parents" we specifically mean that $q(v \vert x_i, c_i, \text{Pa}(v)) = q(v \vert x_i, c_i)$
and $p(v \vert c_i, \text{Pa}(v)) = p(v \vert c_i)$, so it means that it has no parents in $q$,
*conditional on $x$ and $c$*, and no parents in $p$, *conditional on $c$*, which is less restricting than just
"has no parents at all."

This gives us a total ELBO:


$$ELBO = \sum_{\substack{i \in \text{Data} \\ v \in \text{variables}}} ELBO_{i,v}$$

For observed data $v = x$ and $v = c$, the ELBO is a cross entropy.

For generated latent data $v \in V_Z$, the ELBO is a KL divergence.

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
\end{alignat}$$

$$ELBO = \sum_{i \in \text{Data}} ELBO_i$$

With discrete models for $p(x \vert z_1, c)$ and $p(z_1 \vert z_2, c)$, we should end up with a model like this:

![Distribution (x, z, c) ~ q with learned p (discrete)]({{ site.url }}/assets/vae/pt2/qxcz_model.jpeg)

With a gaussian model for $p(x \vert z_1, c)$ and a discrete model for $p(z_1 \vert z_2, c)$, we should end up with a model like this:

![Distribution (x, z, c) ~ q with learned p (gaussian)]({{ site.url }}/assets/vae/pt2/qxcz_model2.jpeg)

It's pretty good, and has good conditional marginals

![Conditional marginal distribution (x, z, c) ~ q with learned p (gaussian)]({{ site.url }}/assets/vae/pt2/qxcz_model2_marginal_c.jpeg)

and overall marginals

![Marginal Distribution (x, z, c) ~ q with learned p (gaussian)]({{ site.url }}/assets/vae/pt2/qxcz_model2_marginal.jpeg)

But if we use gaussian models for both $p(x \vert z_1, c)$ and $p(z_1 \vert z_2, c)$, we get a model like this:

![Distribution (x, z, c) ~ q with learned p (gaussian)]({{ site.url }}/assets/vae/pt2/qxcz_model3.jpeg)

Each component of the model has a decent fit, but we see that errors in generating $z_1$ compound to errors in generating $x$.

It leads to worse conditional marginals

![Conditional marginal distribution (x, z, c) ~ q with learned p (gaussian)]({{ site.url }}/assets/vae/pt2/qxcz_model3_marginal_c.jpeg)

and worse overall marginals

![Marginal Distribution (x, z, c) ~ q with learned p (gaussian)]({{ site.url }}/assets/vae/pt2/qxcz_model3_marginal.jpeg)

This is error compounding a serious weakness in these kinds of models.

# Why is this useful?

This form is really handy for complicated models, as we'll see in the next post covering diffusion.

In general, this form is useful when:
- $q(z \vert x, c)$ and $p(z \vert c)$ can mostly factorize the same way
- We can jointly sample $ (x, c, \text{Pa}(v)) \sim q $
- We can compute $p(v \vert \text{Pa}(v), c)$
- We can compute $q(v \vert \text{Pa}(v), x, c)$ (use bayes rule!)

These criteria guide our choices for variational models.
