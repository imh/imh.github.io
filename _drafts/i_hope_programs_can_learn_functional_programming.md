---
layout: post
title: "I Hope Programs Can Learn Functional Programming"
---

I've been thinking for a while now about how to teach a machine to program.
Most machine learning and statistics is very structured, even the nonparametric stuff.
But we continue to make the same assumptions (like pretending everything's linear) because we have to.
When we try to drop those assumptions, we go to super general models, like trees/forests or deep nets.
These are great, but still mostly work under the assumption that everything lives in some vector space.
For example, plugging a word model into an general-purpose model will typically use something like one-hot encoding with super high dimensions, or word vectors (the super cool [CRFs](http://open.blogs.nytimes.com/2015/04/09/extracting-structured-data-from-recipes-using-conditional-random-fields/ are a neat exception)).

It's certainly not how I think about problem solving or pattern matching, and for general purpose representations, it just seems inefficient.
With enough data and a big enough model we can teach a model to interpolate anything, but extrapolation is still a challenge.
Try going to the [tensorflow playground](http://playground.tensorflow.org/#activation=tanh&regularization=L2&batchSize=15&dataset=spiral&regDataset=reg-plane&learningRate=0.01&regularizationRate=0.003&noise=10&networkShape=6,4&seed=0.80754&showTestData=false&discretize=false&percTrainData=50&x=true&y=true&xTimesY=true&xSquared=true&ySquared=true&cosX=false&sinX=true&cosY=false&sinY=true&collectStats=false&problem=classification&initZero=false) and trying to model the swiss role distribution.

{: .center}
![Swiss roll is hard]({{ site.url }}/assets/swiss_roll_is_hard.png)

Even a really good model isn't going to extrapolate well further from the origin than the training set.
By comparison, it's trivial for us as people to identify the geometric rule actually used to label the data.
Putting that intuition to math, it's something like $$ sin(k r + \theta) > 0 $$.
That rule is much more compact than the one learned by the tensorflow model, and likely would extrapolate far better.
It would be great to be able to teach programs to write rules/programs as compactly as we humans can.

Combining rules together to form other rules is basically programming.
Teaching a machine to program, the first obvious question is what a simple minimal programming environment for a machine would be.
We could go with the most basic building blocks of reads, writes, and memory, like the [NTM paper](https://arxiv.org/abs/1410.5401), but that's seems like the assembly language of program induction.
I can't even write in assembly!
There are a few other approaches out there, but they all still seem very low level, and all imperative.
It occurred to me that with pure functions, I get to be dumb.
I don't have to think about state, and programming is like putting legos together!
Just make sure the type signatures fit and then compose stuff!
Could a machine learn to compose legos?

The challenge here is choice.
Learnable choice is tough since it's not differentiable, but there's been some great success with it in recent years with [attention mechanisms](https://arxiv.org/abs/1406.6247).
If it works for [memory networks](https://arxiv.org/abs/1503.08895), it ought to work here.
Alternatively, reinforcement learning is all about learning choice, and Deepmind has famously been running around showing that it can work for interesting problems.

So I think it could work.
I've spent the last month of weekends and evenings setting out to learn if a simple model of composing the programming version of lego blocks can learn anything interesting.
The first part is deciding what building blocks allow lots of interesting programs to be expressed concisely (hint: lambda calculus is awesome).
The next piece is to represent the AST and function composition in a learnable form.
After that, I'd like to see how to extend beyond simple languages into more expressive ones with user defined data types and fancier type systems until I'm teaching a machine Haskell.
I'm going to try to document the development as I go.