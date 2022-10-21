---
layout: post
title: "Variational autoencoders, part 1: Variational inference and the evidence lower bound"
toc: true
---

Introduce the KL version of VLB and how we can use to expand from one variable to two.
Point out that this KL overestimates support.

Wrap up with how you can expand from one entire space to two.


Variational models are a powerful tool that are often covered unintuitively.
They power some of the most exciting recent advances in generative modeling like DALL-E 2 and Stable Diffusion.

Specifically, I will be talking about variational models with latent variables aka variational autoencoders aka VAEs.

In this four part series, I'll develop the formalism for variational models, emphasizing the intuition behind them.
We will start with the evidence lower bound in this post, and work up to diffusion models.

# Generative models via variational inference

In generative models, you want to learn to sample from some data distribution. Maybe you want to sample cat photos.
Or sci-fi novels. Or higher resolution images that correspond to a given low-res image. Or videos that correspond
to a given text prompt. Whatever it is, it's probably a very messy distribution without a straighforward sampler.

Formally we'll say that we have some true distribution $ q(x) $ that we want to learn, and $ p(x) $ which is our
model of it.

As a toy model, let's say we have data distributed like this:

{: .center}
![Fixed distrubition x ~ q(x)]({{ site.url }}/assets/vae/1/qx.jpeg)

Variational inference is the process finding the best fitting model $ p(x) $ to some observed $ q(x) $, when the class of
model distributions doesn't necessarily include the true distribution $ q(x) $.

We do this by minimizing the KL-divergence between the true distribution $ q(x) $ and the model distribution $ p(x) $

$$ p(x) = \underset{p}{\text{argmin}} D_{KL}(q(x) \Vert p(x)) $$

In the toy model, the true distribution clearly isn't gaussian, but we can find the best fitting gaussian:

![Fixed distribution x ~ q(x) with estimated x ~ p(x)]({{ site.url }}/assets/vae/1/qx_learned_px.jpeg)

It's a terrible approximation. We're dramatically overestimating the support of the distribution and it's obvious that
there's structure to be exploited.

# Adding latent structure

A simple candidate to structure our model is two split it into two groups:

![Fixed marginal distributions x ~ q(x) and z ~ q(z)]({{ site.url }}/assets/vae/1/fixed_qxz_unknown_joint.jpeg)

Denoting the groups as $ z_1 $ and $ z_2 $, we're now trying to see which $ x $ and $ z $ go together.
Ideally we learn a model that connects $ x $ and $ z $ like this, where the dashed lines denote $q(z_j \vert x_i)=1$:

![Fixed distribution (x,z) ~ q(x, z)]({{ site.url }}/assets/vae/1/fixed_qxz.jpeg)

Let's assume we have $ q(z \vert x)$ figured out already (we'll come back to this), grouping the $x$ variables in the
obvious way.

Then we have to build a model of $ p(x, z)$, which we'll factor as $ p(x \vert z) p(z) $.

The simplest candidates are $ p(z) = \text{uniform} $ and $ p(x \vert z) = \text{gaussian} $.

![Fixed distribution (x,z) ~ q(x, z), learned distribution p(x\|z)]({{ site.url }}/assets/vae/1/fixed_qxz_learned_pxz.jpeg)

This is a much better model. **We added a latent variable $z$ to represent the structure of the data, and now
we can get away with a very simple model mapping latent $z$ back to observed $x$.**

# Formalizing it

todo "we'll formalize it with KL divs"

## The evidence lower bound (KL version)

This is where the evidence lower bound (ELBO) comes in:

$$\begin{alignat}{3}
&     && D_{KL}(q(x, z) \Vert p(x, z)) \\
& =   && D_{KL}( q(x) \Vert p(x) ) \\
&     && + \mathbb{E}_{x \sim q(x)} \left[ D_{KL}(q(z \vert x) \Vert p(z \vert x)) \right] \\
& \ge && D_{KL}( q(x) \Vert p(x) )
\end{alignat}$$

We want to minimize $ D_{KL}( q(x) \Vert p(x) ) $, which we effectively do by minimizing
$ D_{KL}(q(x, z) \Vert p(x, z)) $.

In plain terms, we want to minimize the error between our model and data,
which we effectively do by minimizing the error between our expanded model and expanded data.

The ELBO says that if we add in some extra variables $(z)$ to our data and fit a distribution to this expanded space $(x,z)$,
we'll do a decent job fitting a distribution over the original space $x$.

## The evidence lower bound (standard version)

Rewriting $q(x, z)$ as $q(z \vert x)q(x)$ and $p(x, z)$ as $p(x \vert z)p(z)$ we can write the ELBO as:

$$\begin{align}
& D_{KL}(q(x, z) \Vert p(x, z)) \\
&= D_{KL}[q(z \vert x) q(x) \Vert p(x, z) ) \\
&= \mathbb{E}_q \left[ \ln \frac{q(z \vert x) q(x)}{p(x, z)} \right] \\
&= \mathbb{E}_q \left[ \ln \frac{q(z \vert x)}{p(x, z)} \right] + H(q_x) \\
&= \mathbb{E}_q \left[ \ln \frac{q(z \vert x)}{p(x \vert z)p(z)} \right] + H(q_x) \\
&= \mathbb{E}_q \left[ \ln \frac{q(z \vert x)}{p(z)} \right] - \mathbb{E}_q \left[ \ln p(x \vert z) \right] + H(q_x) \\
\end{align}$$

The final term is just the entropy of our data, which is independent of our model $p$, so we can ignore it:

$$ ELBO =  \mathbb{E}_q \left[ \ln \frac{q(z \vert x)}{p(z)} \right] - \mathbb{E}_q \left[ \ln p(x \vert z) \right] $$

**This is the ELBO you'll see in most places.** It's less intuitive, which is why I prefer the KL version.

You can make this form more intuitive by rewriting the first term as KL divergence and the second term as a cross entropy:

$$\begin{alignat}{4}
ELBO =&&  \mathbb{E}_{x \sim q} &\left[ D_{KL} ( q(z \vert x) \Vert p(z)) \right] & \\
  && + \mathbb{E}_{z \sim q} &\left[ CE(q(x \vert z),  p(x \vert z)) \right] &
\end{alignat}$$

But you can't compute that cross entropy because we usually can't easily compute $q(x \vert z)$, so a nice balance is in
between:

$$\begin{alignat}{2}
ELBO = \mathbb{E}_{x \sim q} & \left[ D_{KL} ( q(z \vert x) \Vert p(z)) \right] \\
  - \mathbb{E}_q & \left[ \ln p(x \vert z) \right] 
\end{alignat}$$

It's critical to note that the ELBO is written conditional on our data $x$, meaning that it defines a per-datapoint loss
function:

$$\begin{alignat}{2}
ELBO_i &= D_{KL} ( q(z \vert x_i) \Vert p(z)) - \mathbb{E}_{z \sim q(z \vert x_i)} \left[ \ln p(x_i \vert z) \right] \\
ELBO &= \mathbb{E} \left[ ELBO_i \right]
\end{alignat}$$

# Why is it easier to model $(x, z)$ than to model $x$?

The ELBO says we can't do improve the loss by expanding the space, only upper bound it, so why is this useful?

The power of latent variational models is that we can factor the joint distribution in whatever way is convenient.

For example, we can factor $ p(x, z) = p(x \vert z) p(z) $ where $ p(z) $ is something easy to sample from and
p(x \vert z) is something easy to fit.

Meanwhile, we're stuck with $ q(x) $ as an unknown, but we can make $ q(z \vert x) $ whatever we want, since we're
effectively building ground truth data. All we need is that $ q(z) = \sum_x q(z \vert x)$ is something that $p(z)$
can approximate decently. It's a lot of freedom to get creative.

For example, you could cluster our toy data into two groups and then fit $p$ per cluster, like we did above.

**But you can also learn $q(z \vert x)$ and be safe knowing that minimizing the ELBO also bounds the loss on $x$.**

In the toy example, we can model $p(x \vert z)$ as a gaussian and $p(z)$ as uniform, like before, and simultaneously
model $q(z \vert x)$ as $q(z=1 \vert x) = \sigma(a x)$ with a learned scalar $a$, leaving $q(z=0 \vert x) = 1-q(z=1 \vert x)$.
Then we minimize the ELBO as a function of $q(z \vert x)$, $p(x \vert z)$, and $p(z)$ all at once.

<video muted autoplay loop controls title="Fixed distribution x ~ q(x), learned distributions q(z|x) and p(x|z)">
    <source src="{{ site.url }}/assets/vae/1/learned_qxz_learned_pxz_training.mp4" type="video/mp4">
</video>

We've chained together some incredibly simple models to make those simple models much more powerful.

# What did that get us?

Now we can sample from $ p(x) \approx q(x)$ by sampling from a pair of simple distributions. First we sample
$ z ~ p(z) $, then we sample $ x ~ p(x \vert z) $.

# What's next?

The flexibility of choices for the distributions $q(z \vert x)$, $p(x \vert z)$, and $p(z)$ provide room for creativity,
from something as simple as these toy models to something as complex as text-to-image diffusion models. In the next
posts, we'll dig in to some useful choices for $q(z \vert x)$, $p(x \vert z)$, and $p(z)$, focusing on the diffusion
models that are so popular right now.
