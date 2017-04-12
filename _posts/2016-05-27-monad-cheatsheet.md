---
layout: post
title: "Monad Cheatsheet"
---

I had a [post](http://ian.ai/2016/05/26/why-monads.html) recently about why monads are such a useful abstraction and incidentally introduced Functors too.
I want to reiterate that this is the kind of thing where you have to learn by doing. [These practice problems](https://mightybyte.github.io/monad-challenges/) are what helped me understand this stuff.
But understanding how they work on simple data types can still leave a disconnect to using them in practice.
When I finally began to understand `>>=`, and then saw all the do-notation in the wild, it was confusing, so I thought I'd provide a cheatsheet.

Recall that the type signature of a bind (`>>=`) is

{% highlight haskell %}
(>>=) :: m a -> (a -> m b) -> m b
{% endhighlight %}

To chain these things together into real programs, notice that we can use lambda functions.
In `maybeX >>= f`, `f` has type signature `a -> m a`.
Defining `f` with a lambda, we would write  `maybeX >>= \x -> whatever` (it's the same as `maybeX >>= (\x -> whatever)`).
In this, `x :: a` and `whatever :: m b`.
`whatever` could be any other program that computes an `m b`, and usefully, it now has the unpacked `x` in scope.

I can't underscore how important this is to grasp.
`whatever` can be anything that evaluates to data of type `m b`, and to make it, you can use `x` however you like.
As I pointed out in the last post, the whole reason these things are useful is because you can program without worrying about constantly packing and unpacking your data.
If `whatever` is another monadic statement, like `maybeY >>= \y -> stuff`, then `stuff` is the final value of type `m b`, and it can be computed with both `x` and `y` in scope, but already unpacked for you, whatever that means for your data type.
For `Maybe`, this is just programming with the values if they aren't null, and for lists it's just programming with the elements of the lists, like in a list comprehension.
You didn't have to deal with any of the bug-prone headache of unpacking or repacking `x` and `y`, but now you get to use them the easy way.

That's a bit abstract, so let's make it concrete with an example.
To show how to use this lambda style syntax to build real programs, we just chain them together without having to deal with any `case` statements to handle potential `Nothing`s:

{% highlight haskell %}
justThree =
   (Just 1 >>= \one -> -- one :: Int
    Just 2 >>= \two -> -- two :: Int
    return (one + two))

maybeAdd :: Maybe Int -> Maybe Int -> Maybe Int
maybeAdd mx my =
   (mx >>= \x ->  -- x :: Int
    my >>= \y ->  -- y :: Int
    return (x + y))
{% endhighlight %}

`maybeAdd` is like `justThree`, but can end up `Nothing`.
If `mx` is `Nothing`, it doesn't bother with the right hand side of `>>=` and just returns `Nothing`, as defined in the previous post.
Same for `my`.

Notice that `>>=` just behaves like an overloaded `=` sign that binds to the variable name on the right side.
Hence the name "bind".
Once a variable is bound, you can program with it as usual, but you have to make sure to pack it back up at the end because `>>=` has to return a monadic data structure.
This can be as simple as using the default way of packing things: `return`.

In Haskell, there's an even more convenient syntax for this kind of binding:

{% highlight haskell %}
These are equivalent:
y = (Just 1      >>= \one ->
     Just 2      >>= \two ->
     nullableFun one >>= \x ->
     let y = nonNullableFun two
      in Just (one * y + two * x))

y = do
  one <- Just 1
  two <- Just 2
  x <- nullableFun one
  let y = nonNullableFun two
  Just (one * y + two * x)
{% endhighlight %}

It's entirely equivalent, but makes the assignment nature of "bind" clearer.
Note that `x` is a bound variable representing a nullable thing (monadic), so keeping it in scope downstream is automatic, but `y` is just a plain old value, so you have to use the special `let y = ...` syntax to use it below.

Translating do notation back and forth to bind notation is really good for learning.

{% highlight haskell %}
do
  x <- mX
  ...
{% endhighlight %}

is the same as `do { x <- mX; ...}` is the same as `mX >>= \x -> ...`.
You can translate back and forth between them with just those three things.

One potential point of confusion is when the monadic value (`mX`) isn't bound to anything:

{% highlight haskell %}
do
  mX
  ...
{% endhighlight %}

This is equivalent to `do {mX; ...}` and `mX >>= \_ -> ...`, where it's throwing away the value inside mX.
It still affects the rest of the program, you just can't use it.
For example, if  we're in the `Maybe` monad, and `mX` returns Nothing, the whole thing will return Nothing, even though its `Just x` values wouldn't be in scope downstream.

And because it bears repeating, to really grok this stuff, I'm sorry but you have to learn by doing.
I'm going to plug [these practice problems](https://mightybyte.github.io/monad-challenges/) again because they helped me so much.
If you really want to learn Haskell more generally, I recommend following the progression outlined [here](https://github.com/bitemyapp/learnhaskell).
It's worth it.

Side note: What about Functors?
--------------------

In the previous post, I made a big deal about how these things are convenient because they allow us to program with functions of type take `a -> whatever` instead of `m a -> whatever`, not having to unpack data for ourselves. We can program with raw values, but still overall turn `m a` in to `m b`.
I  tried to show that Functors and Monads both fall out of this goal, but have only shown how to chain Monads together.
Most programs have to work with data from multiple sources, for example using data from two different lists at once.
This post should have made it clear how to do that with monadic data structures:

{% highlight haskell %}
do
  x <- mX
  y <- mY
  ... -- now you can work with both x and y (data from mX and mY)
{% endhighlight %}

This kind of composition is really useful and important.
[The next post](http://ian.ai/2016/06/07/chaining-functors.html) will be about combining Functors together in a similar way.

