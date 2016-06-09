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

The two operations that define a monad are `>>=` (pronounced bind) and `return`.
The hard one is bind.
For the `Maybe` data structure, which is used to wrap nullable data, bind's type signature is

{% highlight haskell %}
(>>=) :: Maybe a -> (a -> Maybe b) -> Maybe b
{% endhighlight %}

In a more java-like notation, this is like

{% highlight java %}
Maybe<B> bind(Maybe<A>, Function<A, Maybe<B>)
{% endhighlight %}

It takes a nullable object of type A, and a function that maps objects of type A to nullable objects of type B, and returns a nullable object of type B.

The reason bind is so convenient is that it's easier to write functions that take an `a`, than it is to write functions that take a `Maybe a`.
The popular alternative would be to write functions of type

{% highlight haskell %}
Maybe a -> Maybe b
{% endhighlight %}

To do this, you have to handle both cases: a normal value and a null value.
This is a pain in the ass and I don't enjoy it.
I'm also prone to introducing bugs here.
It's much easier to just write a function that handles the null value for you once and be done with it.
It returns null if the input is null, and applies your function to a non-null value otherwise.
Simple!

Another example might be lists, where turning a list of xs into ys requires writing an error prone, tedious loop.
Much easier to use a map function!

In the general case, you have some structured data, whether it's `Maybe a`, `List a`, or the variable `f a`, and want to turn them into `Maybe b`, `List b`, or `f b`.
If there's a canonical way to apply a function of type `a -> b`, then `f` is a Functor.
Finding this abstraction is a great way to not repeat yourself.
For Maybe, this is applying the function to the non-null values.
For List, this is the map function.

Sometimes, however, you might want to turn `m a` into `m b` and have a canonical way apply a function of type `a -> m b`.
For Maybe, a null should return a null, and a non-null value should be applied to the function.
For the random number generator monad, it might take a value from one random number generator and use it as a parameter for another random number generator.

These two abstractions, `m a -> (a -> b) -> m b` and `m a -> (a -> m b) -> m b` turn up in all sorts of data structures `m`.
In both cases, being able to program with the raw `a` instead of having to unpack it in cases or a loop saves a ton of time and effort.
When you start using these patterns again and again for your data structure, it's time to factor them out, maybe into the Functor or Monad abstractions, unlocking all of the library functions that go along with them.

The other part of a monad that's really simple is the `return` function whose type is `a -> m a`.
For Maybe, `return x = Just x`.
For List, `return x = [x]`.

Practical Haskell
-----------------

To give an example, Haskell's lambda syntax is useful.
`\x -> f x` defines a function that takes x and applies f to it.
Adding 2 to a `Just 1` could be written as `Just 1 >>= (\one -> return (one + 2))` which, without the unnecessary parentheses is `Just 1 >>= \one -> return (one + 2)`.
The variable `one` has type `Int`, not `Maybe Int`, which is the whole point.

It really starts to pay off in longer programs:

{% highlight haskell %}
Just 1 >>= \one ->
Just 2 >>= \two ->
aNullableFunction one >>= \x ->
return (aNonNullableFunction two) >>= \y ->
return (one * y + two * x)
{% endhighlight %}

Notice that `>>=` just behaves like an overloaded `=` sign that binds to the variable name on the right side.
Hence the name "bind".

In Haskell, there's an even more convenient sugar for this.

{% highlight haskell %}
do 
  one <- Just 1
  two <- Just 2
  x <- aNullableFunction one
  y <- return (aNonNullableFunction two)
  return (one * y + two * x)
{% endhighlight %}

It's entirely equivalent.
Translating do notation back and forth to bind notation is really good for learning.
`do { x <- mX; ...}` is the same as `mX >>= \x -> ...`.
If you don't care about the results, `mX; ...` is the same as `mX >>= \_ -> ...`.

Finally, you probably noticed that `y <- return (aNonNullableFunction two)` just wrapped `aNonNullableFunction two` in a monad only to immediately take it back out.
This is unnecessary, and there's more syntax to use regular variable binding in a do block with plain old `=`:

{% highlight haskell %}
do 
  one <- Just 1
  two <- Just 2
  x <- aNullableFunction one
  let y = aNonNullableFunction two
  return (one * y + two * x)
{% endhighlight %}

That should hopefully provide some motivation around why the bind abstraction is so convenient, and teach you how to translate between it and common Haskell `do` syntax.
And because it bears repeating, to really grok this stuff, I'm sorry but you have to learn by doing.
I'm going to plug [these practice problems](https://mightybyte.github.io/monad-challenges/) again because they helped me so much.
If you really want to learn Haskell more generally, I recommend following the progression outlined [here](https://github.com/bitemyapp/learnhaskell).
It's worth it.

