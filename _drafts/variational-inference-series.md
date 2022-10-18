---
layout: post
title: "Variational inference and diffusion series"
---


pt 1: Variational inference and the evidence lower bound
-------------------------------------------------------

Introduce the KL version of VLB and how we can use to expand from one variable to two.
Point out that this KL overestimates support.

Wrap up with how you can expand from one entire space to two.

pt 2: Diffusion
---------------

Introduce the idea of continuous spaces.
With VLB, you can expand one space into that space plus a continuous space.

You model `(noise <-- x)` as `(noise --> x)`

pt 3: Conditional diffusion
---------------------------

Talk about a few kinds of conditional diffusions.

1) conditional in the estimated model `(noise <-- x)` estimated as `(noise(class) --> x(class))`

2) conditional in the true model: `(noise(class) <-- x(y))` aka `(mean(noise|y) + noise <-- mean(x | y) + x - mean(y))` aka `mean(x|y) + (noise(y)) -> x_delta(y)`. 

Option 2 *should* allow  

pt 4: Targeted conditional diffusion
-----------------------------------

Final kind of conditional diffusion conditions end point (like an initial value problem).

It's walking from one thing to another `(y -> noise -> x)`.

The conditional options above (just conditioning the start point), plus:

1) conditional in the estimated model `(y -> noise -> x)` estimated as `(y -> noise(y, class) -> x(y, class)`

2) conditional in the true model: `y --> noise --> y` aka `(x + 0) --> mean(noise|y) + noise  --> mean(x | y) + x_delta(y))` aka `0 -> mean(x|y) + (noise(y)) -> x_delta(y)`.
