---
layout: post
title: "Implementing Do Notation in Python"
---

## a.k.a. Evil hacks in python for fun ~~and profit~~

**disclaimer:** *This is all a really bad idea, but it was pretty fun to do. Hopefully I can teach people how to do equally horrible-yet-fun things. You should not actually use the decorator I built.*

Monads are a really useful as an abstraction, and a nice clean syntax goes a long way in making them pain-free to use.
In Haskell, that's `do` notation.
Scala has a similar `for` notation.
I spend most of my time in python, though, and it doesn't have anything analogous.
Can we make a pain-free equivalent in python?

We want to find a way to make something that looks nice act like a sequence of lambdas chained to each other with bind statements.
If you squint really hard and turn your head, taking one piece of code and making it do something else is the problem that decorators solve, and as it turns out, you can make a decorator to do pretty much anything in python.
I'll start with a decorator overview for those who haven't used them.
Skip this section if you're comfortable building these already.

# Python decorators

A python decorator is a function that takes a function and returns a function.
I'll use the example of a function that logs the input and return value of another function.

{% highlight python %}
def log_input_and_output(f):
    def sneaky_replacement_function(x):
        print 'INPUT:  ', x
        y = f(x)
        print 'OUTPUT: ', y
        return y
    return sneaky_replacement_function
{% endhighlight %}

This function takes a univariate function `f` and returns another function that just calls `f`, but does other stuff besides (printing its input and output).
We could use it to log some math like so:

{% highlight python %}
def double_x(x):
    return x*2

double_and_log_x = log_input_and_output(double_x)

double_and_log_x(5)  # returns 10
# >> INPUT:  5
# >> OUTPUT: 10
{% endhighlight %}

We took a function `double_x` and passed it to `log_input_and_output`, thereby getting a new function that does all the stuff `double_x` does, and then some.

A lot of times, we aren't actually interested in the function we're defining on its own.
We only ever want the logged version.
In this case we could overwrite it like so:

{% highlight python %}
def double_x(x):
    return x*2

double_x = log_input_and_output(double_x)
{% endhighlight %}

Now we only have the version we want.
Python actually has a special way to do that to make it more convenient:

{% highlight python %}
@log_input_and_output
def double_x(x):
    return x*2
{% endhighlight %}

For almost all purposes, it's equivalent to the code block above, but it looks much nicer and you can chain them together easily.


# Functions of functions

So there you have it: decorators are a nice way of modifying a function you wrote and turning it into a function you didn't.
That's just what we're trying to do to bastardize a new monad syntax!
It sounds close enough, so let's smash this round peg into the square hole.
Let's try to build a decorator that allows something along the lines of:

{% highlight python %}
@with_do_notation
def my_function(x):
    y = do:
        a <- some_monadic_function(x)
        b <- more_stuff(a)
        mreturn(a + b)
    return y
{% endhighlight %}


With any python decorator, we get a function and return a function, but all we're supposed to do is call it in clever ways.
We can call it with different arguments, or do evil things with the output, but in the end, we still just have this black box inside.
What we really need to do to add a new syntax is to break open the black box and muck around with the internals, turning it into valid python before (hopefully) packing it back up again.
Round pegs don't fit into square holes, so we need to whip out our hacksaw.

# We have to stick with valid python

No matter what the decorator `with_do_notation` is, if the function we're decorating isn't syntactically valid python and it's just going to error.
I don't want to write a new language that compiles to python, so I have to obey python's syntax rules.
That means we're searching for valid python that looks like a block of code with some semblance of associating a name to it.
It also can't be something someone's going to accidentally use, since we're overriding its intended meaning.
For binding, I decided to replace haskell's `<-` with plain old equals `=`.
For the block itself, `with x as y:` came to mind:

{% highlight python %}
@with_do_notation
def my_function(x):
    with do as y:
        a = some_monadic_function(x)
        b = more_stuff(a)
        mreturn(a + b)
    return y
{% endhighlight %}

Finally, there's no type inference, so I had to hint out what the type of `mreturn` should be:

{% highlight python %}
@with_do_notation
def my_function(x):
    with do(Maybe) as y:
        a = some_monadic_function(x)
        b = more_stuff(a)
        mreturn(a + b)
    return y
{% endhighlight %}

It's not the prettiest, but it doesn't throw an exception. It's clean enough that I'm happy.

# The ast library is a lovely little hacksaw

Back to our earlier problem, we have to crack open the black box that is `my_function` and rewrite the `with` block inside into a sequence of binds.
For that, I need a way to inspect and modify existing python functions.
Anyone who has abused `__dict__` knows that python is very friendly to this kind of thing.
It's actually scary how friendly this language is to doing things that feel dirty.
Do it too much and you'll start writing blog posts like this one.

For this particular use case, I found the `ast` library.
It gives you tools to work with python code, traversing and modifying the AST as needed.
The [official docs](https://docs.python.org/2/library/ast.html) aren't that great for jumping in, but there's [another set of docs called Green Tree Snakes](https://greentreesnakes.readthedocs.io/en/latest/) that helped me get started.

### Parsing

With the `inspect` and `ast` libraries, we can reconstruct the AST for our `my_function` function:

{% highlight python %}
# inspect.getsource just returns the source code for the function
# textwrap.dedent unindents it, so we can parse it alone
src = dedent(inspect.getsource(my_function))
# Use the ast library to parse the function definition
module = ast.parse(src)
# pretty print the module with this: https://bitbucket.org/takluyver/greentreesnakes/src/default/astpp.py?fileviewer=file-view-default
# "print dump(module)" returns this:
Module(body=[
    FunctionDef(name='my_function', args=arguments(args=[
        Name(id='x', ctx=Param()),
      ], vararg=None, kwarg=None, defaults=[]), body=[
        With(context_expr=Call(func=Name(id='do', ctx=Load()), args=[
            Name(id='Maybe', ctx=Load()),
          ], keywords=[], starargs=None, kwargs=None), optional_vars=Name(id='y', ctx=Store()), body=[
            Assign(targets=[
                Name(id='a', ctx=Store()),
              ], value=Call(func=Name(id='some_monadic_function', ctx=Load()), args=[
                Name(id='x', ctx=Load()),
              ], keywords=[], starargs=None, kwargs=None)),
            Assign(targets=[
                Name(id='b', ctx=Store()),
              ], value=Call(func=Name(id='more_stuff', ctx=Load()), args=[
                Name(id='a', ctx=Load()),
              ], keywords=[], starargs=None, kwargs=None)),
            Expr(value=Call(func=Name(id='mreturn', ctx=Load()), args=[
                BinOp(left=Name(id='a', ctx=Load()), op=Add(), right=Name(id='b', ctx=Load())),
              ], keywords=[], starargs=None, kwargs=None)),
          ]),
        Return(value=Name(id='y', ctx=Load())),
      ], decorator_list=[]),
  ])
{% endhighlight %}

That's the AST for `my_function`.
Its bulky, but you can recognize its original definition from above in it.
We can work with that.

# Rewriting the AST

The goal here is to find any `With` nodes and rewrite their contents.
`ast` makes this super easy with `ast.NodeTransformer`.
If you inherit from `NodeTransformer`, you can override the `visit_Foo` method to do something special to the `Foo` nodes.
In this case, we want to override the `With` block:

{% highlight python %}
class RewriteWithDo(NodeTransformer):
    def visit_With(self, node):
        self.generic_visit(node)
        # Make sure its context expression is a function called "do"
        if not (hasattr(node.context_expr, 'func') and
                node.context_expr.func.id == 'do'):
            return node
        name = node.optional_vars.id
        # The argument of the "do" function is the name of the monad class.
        monad = node.context_expr.args[0].id
        bind_chain = rewrite_with_to_binds(node.body, monad)
        # Assign the result of the bind chain to the name in
        # with do(...) as name:
        return Assign(targets=[Name(id=name, ctx=Store())],
                      value=bind_chain)
{% endhighlight %}

We check to make sure we're only rewriting nodes for the `With` blocks of the form `with do(MyClass) as my_name:`, then rewrite the body into a sequence of monadic binds:

{% highlight python %}
class RewriteDoBody(NodeTransformer):
    def __init__(self, monad):
        self.monad = monad
        super(RewriteDoBody, self).__init__()
    def visit_Call(self, node):
        self.generic_visit(node)
        if not (isinstance(node.func, Name) and
                node.func.id == 'mreturn'):
            return node
        node.func = Attribute(value=Name(id=self.monad, ctx=Load()), attr='mreturn', ctx=Load())
        return node
    # TODO allow let bindings in do block

def rewrite_with_to_binds(body, monad):
    new_body = []
    # Construct a transformer for this specific monad's mreturn
    rdb = RewriteDoBody(monad)
    # This is the body of the lambda we're about to construct
    last_part = body[-1].value
    # Rewrite mreturn
    rdb.visit(last_part)
    # Iterate in reverse, making each line the into a lambda whose body is the
    # rest of the lines (which are each lambdas), and whose names are the
    # bind assignments.
    for b in reversed(body[:-1]):
        rdb.visit(b)
        if isinstance(b, Assign):
            name = b.targets[0].id
            value = b.value
        else :
            # If there was no assignment to the bind, just use a random name, eek
            name = '__DO_NOT_NAME_A_VARIABLE_THIS_STRING__'
            value = b.value
        # last part = value.bind(lambda name: last_part)
        last_part = Call(func=Attribute(value=value, attr='bind', ctx=Load()),
                         args=[Lambda(args=arguments(args=[Name(id=name, ctx=Param()),],
                                                     vararg=None, kwarg=None, defaults=[]),
                                      body=last_part),],
                         keywords=[], starargs=None, kwargs=None)
    return last_part
{% endhighlight %}

With these node transformers, we can finally write our decorator:

{% highlight python %}
def with_do_notation(f):
    # Get the context for the function we're calling this from
    frame = inspect.stack()[1][0]
    # Get the function's source
    src = dedent(inspect.getsource(f))
    # Parse it into an AST
    module = parse(src)
    function_def = module.body[0]
    function_name = function_def.name
    assert(isinstance(function_def, FunctionDef))
    # Rewrite any `with do(MyMonadInstance) as my_name:` blocks
    RewriteWithDo().visit(module)
    # Remove the with_do_notation decorator, so it doesn' recurse infinitely
    # when we compile it
    function_def.decorator_list = [d for d in function_def.decorator_list
                               if not (isinstance(d, Name) and d.id=='with_do_notation')]
    # Define the function in the scope it was originally defined, with its
    # original name and new body
    exec(compile(fix_missing_locations(module),
                 filename='<ast>', mode='exec'), frame.f_globals, frame.f_locals)
    # Return the new function
    return eval(function_name, frame.f_globals, frame.f_locals)
{% endhighlight %}

There it is!
A neat decorator to implement totally new and abusive functionality in about 50 lines (plus comments).
You can find the whole thing [here on github](https://github.com/imh/python_do_notation/blob/master/do_notation.py).

# Examples

I also wrote up some examples in that repo to see whether it works for decently and is actually painless to use.

[I implemented the maybe monad](https://github.com/imh/python_do_notation/blob/master/maybe_example.py), so you can write code like this:

{% highlight python %}
just = lambda x: Maybe(just=x)
nothing = Maybe()

@with_do_notation
def decrement_positives(x):
    with do(Maybe) as y:
        a = just(x) if x > 0 else nothing
        just(a-1)
    return y

print decrement_positives(0)  # Nothing
print decrement_positives(1)  # Just 0
print decrement_positives(2)  # Just 1
{% endhighlight %}

[I implemented the list monad](https://github.com/imh/python_do_notation/blob/master/list_example.py), so you can write code like this:

{% highlight python %}
@with_do_notation
def list_example():
    list1 = ListMonad([1,2,3])
    list2 = ListMonad([-1,2])
    with do(ListMonad) as z:
        x = list1
        y = list2
        mreturn(x*y)
    assert(sorted(z.lst) == sorted([x*y for x in list1.lst for y in list2.lst]))
    return z

print list_example()  # ListMonad([-1, 2, -2, 4, -3, 6])
{% endhighlight %}

And finally, for a richer test, [I implemented a monadic parser](https://github.com/imh/python_do_notation/blob/master/parser_example.py).
It was tremendously satisfying and was a total breeze to write with this notation.
I got to write code like this:

{% highlight python %}
@with_do_notation
def parse_string(string):
    if string == "":
        return Parser.mreturn("")
    if len(string) > 0:
        with do(Parser) as x:
            x = char(string[0])
            parse_string(string[1:])
            mreturn(string)
        return x
    return parser_zero

@with_do_notation
def sepby1(p, sep):
    with do(Parser) as throwaway_a_sep:
        sep
        p
    with do(Parser) as p2:
        x = p
        xs = many(throwaway_a_sep)
        mreturn(x + xs)
    return p2
{% endhighlight %}

Of course, this kind of recursive programming will exceed your max recursion depth pretty quickly, but it was great to see that, at least in principle, it's not too hard to make it easy.
