---
layout: post
title: "Teaching Programs to Program is Hard"
---

Towards the beginning of the summer, [I wrote up a post](http://ian.ai/2016/06/08/i-hope-programs-can-learn-functional-programming.html) on how I think we could teach programs to program.
The gist is that there's a small-ish well typed program space worth exploring in pure functional programming.
Pure strongly typed FP can be really restrictive in what programs it allows, which means a machine has fewer bad programs it has to try out.

I worked on it for a few months, but abandoned it around June or July.
The next steps to improving it were big ones and, because it's just a fun side project, they're too big to be worthwhile.
As such, I'm putting it on hold.
That said, I learned a ton that might be worth sharing and wrote code people might be interested in, so here's a little writeup.
Maybe I'll add more later, but don't hold your breath.

The first steps I took were purely proof of concept.
Can a neural network using basic RL learn to choose between alternative programs for this kind of problem?
I set out to test this with a series of toy problems.

# Addition or multiplication?

The first test was to learn whether a bunch of numbers were summed or multiplied.
Given a list of pairs $$(x_{1i}, x_{2i})$$ and associated $$y_i = x_{1i} \oplus x_{2i}$$ where $$\oplus$$ is either addition or multiplication, I wanted to see whether a simple policy gradient algorithm could learn which gives smaller errors?
I did this with policies that chose addition with probability $$sigmoid(\theta)$$ and multiplication otherwise, searching for a policy that minimized the error.

The most basic way to do policy gradient is to notice that $$\nabla_\theta \mathbb{E}_X[f(x)] = \mathbb{E}_X[f(x) \nabla_\theta \textrm{ln}p(x)]$$.
This does ok, but the finite sample analogue is kinda crappy.
Realizing that the gradients should be the same if we subtract some baseline $$\nabla_\theta \mathbb{E}_X[f(x) - b]$$, we can surprisingly decrease the variance of our gradient estimates by choosing a non-zero baseline.
Simply setting the baseline to be $$\mathbb{E}_X[f(x)]$$, leading to gradient estimates of $$\mathbb{E}_X[\{f(x) - \mathbb{E}_X[f(x)]\} \nabla_\theta \textrm{ln}p(x)]$$ does a much better job.

Simple gradient descent on $$\theta$$ chooses the correct branch pretty much immediately.

# Addition or multiplication by what?

The next test was a little harder.
This time it had to learn $$y_i = x_i \oplus k$$ for unknown operation $$\oplus$$ and constant $$k$$.

This one was harder.
Schulman et al explain really well in their [stochastic computation graph paper](https://arxiv.org/abs/1506.05254) that a sampling step really just screws your gradients.
This is tough when you have continuous parameters that the loss function *should* be cleanly differentiable with respect to.
To make it fit into this kind of framework, I had to be able to sample $$ k $$, so I chose to model it as a normal distribution with unknown mean and log variance.
It would have been easier to use a different parameter for $$k$$ per branch of the $$\oplus$$ choice, cutting out the difficulties around sampling, but the whole point was to do a proof of concept on learning shared global weights in the simplest model.

The results were really interesting.
Once I switched from the most basic policy gradient algorithm to the one with a baseline, it was able to learn the model pretty well most of the time, but not always.
For one thing, the variance parameter decreased way too fast.
If I forced it to stay artificially high, it would find the true value far more reliably, and allow the $$ \oplus $$ time to find the right value as well, but it wouldn't learn $$ k $$ nearly as precisely.
It was also pretty sensitive to parameter initialization.

With some tuning, it could learn quickly and precisely, so I moved on to a harder problem.

# Simply typed lambda calculus

Yay!
The problem I wanted to do from the beginning!
I treated it as a tree to sample from, with a stateful search recursing through the tree with an LSTM.
Given an input variable `x` of type `a` and output variable `y` of type `b`, we can construct a candidate `y` by applying a lambda of type `a -> b` to `x`.
This passes the buck up a step.
What lambda should we construct?

Well, a lambda has two fields in it.
The first is a variable (in this case a variable of type `a`), and the second is an expression of type `b` (with the new variable in scope).
Each of these fields is is constructed recursively:

Each language rule can construct something, like an expression, variable, or primitive value of a given type.
Many of the rules also have fields, like the a lambda's variable and body fields.

The rules are also assigned embeddings.
When trying to sample a field for a given rule, we need to give it a probability.
We can construct unnormalized probabilities with some function $$ rule\_prob(state, rule\_embedding) $$.
We then sample from the valid rules that could make an expression of the desired type.
We use that rule to make an expression/variable/primitive, constructing its own fields recursively, if it has any
Eventually, the fields chosen are variables or primitives, which don't have fields of their own.

For each field, we modify the state as a function of the rule that was chosen:
$$ new\_state = state\_update(state, rule\_embedding) $$.
This lets the state represent the parts of the AST that have been created so far, and helps the model intelligently choose the rest of the AST.

In the end, we end up with an AST of the correct type, and a differentiable probability of that AST with respect to our model parameters.

We then apply the function $$f$$ from our AST to each $$x_i$$ and evaluate it, from which we can construct a not-necessarily-differentiable loss function $$loss(f(x_i), y_i)$$ and it's probability.
Applying reinforcement learning, we can find the parameters that make the true AST most likely to be generated.

If we sampled enough ASTs, we ought to be able to find the right one, and increase its probability, or at least increase the probabilities of better approximations to the true one.
That's a rough description, and the actual code is [here](https://github.com/imh/func_learn/tree/master/src/mono).

It turns out that in the space of all ASTs in a simple language, most of them are huge, in the same way that most positive integers are over a million.
Who would have thought!?
At first, I thought I had a bug when I couldn't generate a single AST, but then I realized it just took forever.
Penalizing samples that overflowed some small depth, it was able to run quickly enough.

# Finally, results!

It didn't take too horribly long to learn simple functions like `lambda x: True`, `lambda (x1, x2): x1`, or `lambda (x1, x2): (x1 * x1) + x2` or other "programs" of similar complexity, but this is where I realized I was going down the wrong path.
First of all, it took longer than I'd like to go from most most samples overflowing the maximum tree depth, to most samples being small enough to evaluate.
Second, it took a huge number of iterations between first seeing the best AST and making it the most probably AST.
In between, it would sample the same trees thousands of times with the same loss but different probabilities.
Finally, with finite samples, it was really easy to get stuck in local minima, never seeing function the model ought to make more likely.

# Ouch

The simplest change would be to stop sampling from the possible trees and instead turn it into a search problem where we only see each possible tree at most once per gradient step.
Importance sampling (or something like 12.5 of [Koller and Friedman](http://pgm.stanford.edu/)) might not be the right idea, but they seem like the right flavor.
I don't actually *need* an unbiased estimate of the gradient.
All I really need is a way to intelligently decide where to search next in the tree of all possible ASTs.
Gradients of stochastic policies seem like total overkill in retrospect.

The problem there is that I'd have to write this whole thing as an infinite tree of trees to explore, and that just seem like a pain in the butt.

Alternatively, moving to imperative-land could be promising.
Given a list of variables $$x_i$$ in scope, try to turn each into $$y_i$$ by continuously manipulating them (in parallel) until they seem correct.
All operations would have to happen uniformly across all $$i$$, so it couldn't cheat much worse than just returning $$mean(y)$$ as a constant for all $$i$$.
This "hack at it until it works" approach helps me think through new problems sometimes, so maybe it's a decent attempt.
It strikes me as an easier problem in a larger solution space, so who knows if it'll be easier for a machine to find. ¯\\\_(ツ)\_/¯

But both of those are too much work for a side project, so this is on hold for a while. :(

# What I learned

My takeaway from this whole project was that, when searching through huge/infinite spaces, an approach that samples individual points redundantly is a waste.
Maximizing likelihood that good solutions are found doesn't require that we start the search all over for every parameter update.
The focus shouldn't be on maximizing the likelihood of those points given that they are found, the focus should be on finding them in the first place.
