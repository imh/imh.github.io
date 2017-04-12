---
layout: post
title:  "Why Monads?"
---

What better way to start a blog than Yet Another Monad Tutorial?
Lots of the other tutorials try to explain what a monad is, and that's extremely hard.
I'm hoping to help by motivating them and talk about why monads are such a *convenient* abstraction to work with.
To really get the hang of them, though, there's really nothing better than doing.
[These practice problems](https://mightybyte.github.io/monad-challenges/) are fantastic.
In this post, I will assume knowledge of [basic haskell syntax](https://prajitr.github.io/quick-haskell-syntax/).

For most of the post I will use the `Maybe` datatype, which is just a data type that can also be null. A variable of type `Maybe a` can either be `Just x` where `x :: a` or `Nothing`. 

The painful part of working with data types like this is having to unpack them all the time.
If you want to chain methods together with a function like `(&)`, it can get really painful:

{% highlight haskell %}
(&) :: a -> (a -> b) -> b
x & f = f x

pureFun :: a -> b
pureFun = -- some piece of business logic

nullableFun :: a -> Maybe b
nullableFun = -- some piece of business logic that might return "Nothing"

y = Just x &  -- x :: a
    doSomething &
    doSomethingElse
  where
    doSomething mVal =
      case mVal of
          Just val -> Just (pureFun val)  -- pureFun :: a -> b
          Nothing  -> Nothing
    doSomethingElse =
      case mVal of
          Just val -> nullableFun val  -- nullableFun :: b -> Maybe c
          Nothing  -> Nothing
{% endhighlight %}

That's a lot of packing and unpacking.
That kind of programming isn't just a pain in the butt, it's where I introduce the most common bugs.
It would be much better if we could just abstract out all the boilerplate packing/unpacking of our data.
Following the pattern above, it would be something like this:

{% highlight haskell %}
maybeApply :: Maybe a -> (a -> ???) -> Maybe b
maybeApply packedValue f =
  case packedValue of
    Just unpackedValue -> ???
    Nothing            -> Nothing
{% endhighlight %}

The `???` pieces depend on exactly what we're doing, but the rest is going to be the same no matter what.
Looking at the example above, the functions we're applying have two different types:

{% highlight haskell %}
pureFun     :: a -> b
nullableFun :: a -> Maybe b
{% endhighlight %}

Throwing those into our maybeApply would give us two different kinds of functions

{% highlight haskell %}
maybeApplyPure :: Maybe a -> (a -> b) -> Maybe b
maybeApplyPure packedValue pureFun =
  case packedValue of
    Just unpackedValue -> Just (pureFun unpackedValue)
    Nothing            -> Nothing

maybeApplyNullable :: Maybe a -> (a -> Maybe b) -> Maybe b
maybeApplyNullable packedValue nullableFun =
  case packedValue of
    Just unpackedValue -> nullableFun unpackedValue
    Nothing            -> Nothing
{% endhighlight %}

The first case handles all functions that are guaranteed to return useful data, and the second handles functions that might return `Nothing`.
With those in hand, we could rewrite our original code as

{% highlight haskell %}
y = Just x `maybeApplyPure`
    pureFun `maybeApplyNullable`
    nullableFun
{% endhighlight %}

The only functions we have to worry about here are the ones that handle actual data.
We can spend our mental effort (and debugging effort!) on whatever `pureFun` and `nullableFun` happen to be, instead of worrying about the containers over and over.
It's a much better way to program.

We can generalize this to more generic containers too:

{% highlight haskell %}
-- These:
maybeApplyPure         :: Maybe a -> (a -> b)       -> Maybe b
maybeApplyNullable     :: Maybe a -> (a -> Maybe b) -> Maybe b
-- are replaced by these:
containerApplyUnpacked :: f a     -> (a -> b)       -> f b
containerApplyPacked   :: f a     -> (a -> f b)     -> f b
{% endhighlight %}

The implementation depends on the datatype `f`, of course, but they tend to be really useful.
Worry about packing and unpacking your container as rarely as you can.
The rest of the time, just write code that deals with the data inside.
In my mind, that's why Monads and functors are so useful.
The type signatures for the methods that are hardest to think about for a newbie are:

{% highlight haskell %}
(flip fmap) :: (Functor f) => f a -> (a -> b)   -> f b
(>>=)       :: (Monad f)   => f a -> (a -> f b) -> f b
{% endhighlight %}

These are just the `maybeApplyPure`/`containerApplyUnpacked` and `maybeApplyNullable`/`containerAppplyPacked` we derived above. (`fmap` has its arguments reversed).

The other part of a monad is just a really simple function

{% highlight haskell %}
return :: (Monad m) => x -> m x
-- Example for maybe
return :: x -> Maybe x
return value = Just value
{% endhighlight %}

`return` is just a "default" way of packing a single piece of data into your data structure.

There's a bit more to it than that, regarding monad laws and functor laws that must be obeyed to really be a monad, but intuitively, monads and functors are just ways of codifying how to deal with data inside a data structure, depending on whether the intermediate code returns data that is itself in a container or not.

Lists, for example, are one of the other most common data structures.
A functor for a list of type `[a]` would just define a way to apply a function `f` of type `a -> b`, turning it into another list of type `[b]`.
The most obvious way to do that would be to turn each `a` into a `b` by apply `f` to each one.
A monad for a list of type `[a]` also turns it into `[b]`, but this time, by applying a function `f` of type `a -> [b]`.
The most obvious way to do this is to just apply `f` to each element, and concatenate the resulting lists together.

That's the main idea.
Just like you don't want to deal with the pain of for loops and array index out of bounds errors, you don't want to manually unpack your data every time you use it.
Some of the most generic functions to use on generic containers are defined by functors and monads.
By defining functions for generic containers instead of having different ones for Maybe and List, we can build up more powerful tools that apply to most any data structures with canonical ways of unpacking and applying functions.
In the next posts, I'll get pragmatic and talk about [the syntax that makes them easier to use](http://imh.github.io/2016/05/27/monad-cheatsheet.html), and [how to compose these things together](http://imh.github.io/2016/06/07/chaining-functors.html).
