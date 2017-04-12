---
layout: post
title: "Chaining Functors"
---

[A couple posts ago](http://imh.github.io/2016/05/26/why-monads.html), I made a big deal about how Monads and Functors are convenient because they allow us to program with functions that take `a -> whatever` instead of `m a -> whatever`, not having to unpack data for ourselves, but still overall turn `m a` in to `m b`.
I  tried to show that Functors and Monads both fall out of this goal of working with the data inside a data structure without having to manually unpack and pack it over and over again.
In [my last post](http://imh.github.io/2016/05/27/monad-cheatsheet.html), I showed how to use this to compose monads, combining them together to make complex programs in a convenient way.
What about functors?
If they are just as convenient as monads, can we chain functorial computations together to write complex programs too?
The answer is yes and no.

Given a bunch of functions `g :: a -> b` and `h :: b -> c`, it's easy to use `fmap :: f a -> (a -> b) -> f b` to turn `fX :: f a` into an `f c`.
We just use the usual function composition `fmap fX (h . g)`.
In that sense, they're easy to chain.

But most useful programs aren't as simple as composing single variable functions together.
We have things coming in from various sources and we have to combine them and split them apart.

For monads, we apply a multivariate function `a -> b -> c` to `m a` and `m b` by doing

{% highlight haskell %}
-- for a concrete example
maybeAddition :: Maybe Int -> Maybe Int -> Maybe Int
maybeAddition mX mY =
  mX >>= \x ->
  mY >>= \y ->
  return (x + y)
-- and the general case:
funApply :: m a -> m b -> (a -> b -> c) -> m c
funApply mX mY fun =
  mX >>= \x ->
  mY >>= \y ->
  return (fun x y)
{% endhighlight %}

First we have to get the unpacked version of `mX` and `mY` into scope and then we can apply `fun`.

For functors, it's not so simple. Let's give it a shot the same way we did for monads:

{% highlight haskell %}
(<$$>) :: f a -> (a -> b) -> f b
x <$$> f = fmap f x -- just a flipped fmap to more closely match >>=

funApply :: f a -> f b -> (a -> b -> c) -> f c
funApply fX fY fun =
  fX <$$> \x ->  -- fX :: f a, and so x :: a from the definition of <$$>
  whatever       -- what's the type of whatever?
{% endhighlight %}

We run into a problem.
From the definition of `<$$>`, we see that if `whatever :: d`, then the final result of `(fX <$$> \x -> whatever) :: f d`.
From our funApply type signature, we want that to be `f c`, so we must have `whatever :: c`.
The problem is that we don't have a `y :: b` from the `fY :: f b` in scope yet. Oh no!

With monads, we get `>>= :: m a -> (a -> m b) -> m b`. `whatever` has type `m b` in `mX >>= \x -> whatever`, so we can do more monadic unpacking inside it to unpack more things, like `whatever = mY >>= \y -> stuff`, where we still end up with `stuff :: m b`.

Because fmap has to take a pure function `(a -> b)`, `fX <$$> \x -> fWhatever` requires that `fWhatever` has type `b`. We aren't in a packed datastructure anymore and are limited to a very particular kind of computation.
We can't take `f b` out too.

So what can we do? Let's follow the types out and see what we need. For a bivariate function `fun :: a -> b -> c` we could try the following:

{% highlight haskell %}
(<$$>) :: f a -> (a -> b) -> f b
x <$$> f = fmap f x -- just a flipped fmap to more closely match >>=

-- We know how to unpack one value fX, so let's just do that
funApply :: f a -> (a -> b -> c) -> f (b -> c)
funApply fX fun =
  fX <$$> \x ->
  fun x
-- this is just "fmap fun fX"
{% endhighlight %}

Are we any closer? Now we have a `f (b -> c)` in we can use, but there's no clear way to apply that to a `f b`.

To make it semi-concrete again, let's go back to `Maybe`. A "function" of type `Maybe (b -> c)` is either just a function `Just fun` or `Nothing`. It seems pretty clear how we ought to apply `Just fun` to `Just x` or `Nothing`, and applying `Nothing` should just be `Nothing`, so maybe `Maybe (b -> c)` isn't such a crazy kind of function to apply.

More generally, if we define how to apply functions `f (b -> c)` to `f b`, then we could apply this weird half-applied, sorta packaged up function (or package full of functions) to our data structures.
Turns out that defining that packaged function application operation on a functor defines an applicative functor (just called Applicative).
The operation is denoted `<*>`: `(<*>) :: Applicative f => f (a -> b) -> f a -> f b`.

With that, we can finally define this kind of complex chaining for pure functions to use `a -> b -> c` on `f a` and `f b`.
`fmap fun fX` gives us a `f (b -> c)` and `<*>` lets us apply it to `fY :: f b`.
It's just `(fmap fun fX) <*> fY`.
Applicative actually defines an infix `fmap` written `<$>`, which makes it even easier to write: `(fmap fun fX) <*> fY == (fun <$> fX) <*> fY == fun <$> fX <*> fY`.
Because haskell is all about partially applying functions, it's just as simple to keep applying this to more and more arguments:

{% highlight haskell %}
funXYZ :: a -> b -> c -> d
fX :: f a
fY :: f b
fZ :: f c
funXYZ <$> fX <*> fY <*> fZ :: f d
-- from left to right:
funXYZ <$> fX :: f (b -> c -> d)
funXYZ <$> fX <*> fY :: f (c -> d)
funXYZ <$> fX <*> fY <*> fZ :: f d
{% endhighlight %}

The motivation is the same as that for normal functors.
You don't want to manually unpack your data, and want to use pure functions.
It just takes a little extra defining to let you do it for multivariate functions.

There's actually a GHC extension called ApplicativeDo that lets you write this chaining with do notation, binding variables and everything.
I'm not 100% sure on the syntax, but I think the translation is as follows:

{% highlight haskell %}
do
  x <- fX
  y <- fY
  pure (f x y) -- pure is just like Monad's return.

-- is the same as
f <$> fX <*> fY
{% endhighlight %}

It's sort of cheating to say that Applicative is how you combine multiple functors, but that's my intuition.

Finally, it turns out that functors, applicatives, and monads form a hierarchy.
All monads are applicatives and all applicatives are functors.
Personally, I find it more obvious when and how to define a functor and monad instance for my data types than for appicatives.
For functors, it's just how to you want to apply functions to data that can come out of your data structure.
For monads, it's the same, but for when you want to apply functions that themselves return structured data.
For applicatives, the only intuition I can offer is that of combining functors together.
