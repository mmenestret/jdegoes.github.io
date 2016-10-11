---
layout:       post
title:        "Descriptive Variable Names: A Code Smell"
description:  "Polymorphic code is more likely to be correct than monomorphic code"
category:     articles
tags:         [fp, functional programming, haskell, purescript, polymorphism, monomorphism, names]
---

Descriptive variable names are a code smell.

More precisely, if you can name your variables after more descriptive things
than `f`, `a`, `b`, and so on, then your code is probably *monomorphic*.

Monomorphic code is much more likely to be incorrect than polymorphic code,
because for every type signature, there are many more possible implementations.

Thus, descriptive variable names are a code smell, indicating your code is
overly monomorphic and more likely to be broken.

Fortunately, you can turn this around with a little refactoring process I
call, *extract polymorphism from monomorphism*. While this refactoring does not
always eliminate monomorphism, it can *defer* it, which is an overriding theme
of functional programming (defer all the things!).

In the next section, I'll show you this refactoring in action.

## Monomorphism in String Functions

Some functions are maximally monomorphic &mdash; which is to say, every type in
the signature is *concrete*, and there are no values in the signature described
by type parameters.

For such monomorphic functions, the "number" of possible implementations is
exceedingly great: we can implement them in numerous ways that satisfy the type
signature, but are horribly broken.

For example, consider how many ways there are to implement the following
function:

{% highlight haskell %}
foo :: List Char -> List Char -> List Char
{% endhighlight %}
{% highlight scala %}
def foo(string1: List[Char], string2: List[Char]): List[Char]
{% endhighlight %}

There are a huge number of ways to implement the function. The type signature of
the function barely constrains it. All we know is that if we feed the function
two strings, we'll get back a string.

The string may or may not be related to either input. It may contain random
hard-coded characters. Or it might be empty.

Now consider how many ways there are to implement the following function:

{% highlight haskell %}
foo :: forall a. List a -> List a -> List a
{% endhighlight %}
{% highlight scala %}
def foo[A](fst: List[A], snd: List[A]): List[A]
{% endhighlight %}

This function is at least *somewhat* polymorphic. There are fewer ways we can
implement the function. In particular, we can't just hard-code some elements in
a list, because we have no ability to manufacture values of an arbitrary type.

Adding a type parameter to describe the type of the elements in the list has
constrained our space of possible solutions!

However, there's still lots of ways to implement the function: we can return
the first list, return the second list, return some mashup of the first and
second lists, or return an empty list.

Let's say we take this one step further and introduce even more polymorphism
into the code, hiding the fact that the second parameter and return values are
lists:

{% highlight haskell %}
foo :: forall a b. List a -> (a -> b) -> (b -> b -> b) -> b -> b
{% endhighlight %}
{% highlight scala %}
def foo[A, B](as: List[A], b: B, ab: A => B, bbb: B => B => B): B
{% endhighlight %}

There are considerably fewer ways to implement this function. The function can
return `b` (a useless definition, since it doesn't use the type parameters), or
it convert `a`'s to `b`'s, and smash `b`'s together to produce a final `b`.

If we continued this process to its logical conclusion, and leverage standard
type classes, we might end up with the following types:

{% highlight haskell %}
foo :: forall f a r. (Foldable f, Semigroup r) => f a -> (a -> r) -> r -> r
{% endhighlight %}
{% highlight scala %}
def foo[F[_]: Foldable, A, R: Semigroup](fa: F[A], ar: A => R, r: R): R
{% endhighlight %}

This is a sufficiently high degree of polymorphism that our variable names are
now totally abstract, named after their types.

It's high enough I'm betting you can *guess* what the function `foo` is supposed
to do (let me know if you need a hint!).

Note that in our process of trying to polymorphize this function, we lost the
original type signature, which was, presumably, required by a bunch of code.

Our program may not need the original type signature, if we can force the
polymorphism higher into the code base.

However, if our code *does* require the original type signature, then we can
recapture the original definition quite simply:

{% highlight haskell %}
foo0 :: List Char -> List Char -> List Char
foo0 l1 l2 = foo l1 pure l2
{% endhighlight %}
{% highlight scala %}
def foo0(l1: List[Char], l2: List[Char]): List[Char] =
  foo[List, Char, List[Char]](l1, List(_), l2)
{% endhighlight %}

While these signatures are the same as the old ones, their implementation is
trivial, and very easy to reason about.

In essence, what we've done is created a little *bubble of polymorphism* in
which we pushed as much logic as possible, so that the amount of monomorphic
code we have to reason about is as small as possible.

Of course, in many cases, you can push the polymorphism even higher, by looking
at the functions that require `foo` and generalizing them.

This is just a toy example, so I'll briefly run through a real-world example to
show how the technique works in the wild.

## Real-World Example: Grouping

A few months ago, I was working on a function that had this type signature:

{% highlight haskell %}
type JoinMap = Map (Set Data) (List Data)
groupByJoinKeys :: List (List Data -> Data) -> List Data -> JoinMap
{% endhighlight %}
{% highlight scala %}
type JoinMap = Map[Set[Data], List[Data]]
def groupByJoinKeys(joinKeys: List[List[Data => Data]], dataset: List[Data]): JoinMap
{% endhighlight %}

The purpose of the function was to iterate through a `dataset`, and apply a
bunch of projections to the data (the "join keys"), and then stuff the values of
the dataset into a map keyed off the projections.

I don't recall my first implementation of this function, but I know for a fact
it was broken. *Totally broken*, in fact, and unnecessarily complex.

After a bit of work, I managed to extract out something much more polymorphic:

{% highlight haskell %}
groupBy :: forall k v. (Ord k) => (v -> k) -> List v -> Map k (List v)
{% endhighlight %}
{% highlight scala %}
def groupBy[K: Ord, V](vk: V => K, vs: List[V]): Map[K, List[V]]
{% endhighlight %}

There's still lots of ways to implement this function, but far fewer than the
original definition.

Although I could have taken the technique farther, in this case, I stopped here
(the point of diminishing returns). A correct implementation proved straightforward.

Finally, I went back to `groupByJoinKeys`, and implemented this monomorphic
function in terms of the polymorphic `groupBy` &mdash; because in this case, I
couldn't easily push the polymorphism any higher.

The implementation of `groupByJoinKeys` turned out to be a simple one-liner.

## Nothing New Under the Sun

The technique I've elaborated here is not unique to functional programming.

In object-oriented programming, we had a saying: *require the least powerful
interface you need to implement a method*.

In other words, if your method only needs to know if a thing is a *Shape*, then
don't also require it to be a *Hexagon* (a subtype of *Shape*).

This technique applies to functional programming, as well, only since we don't
typically have subtyping, we use polymorphism, and when that alone is
insufficient, we add type class constraints to provide more structure.

Whether in OOP or FP, the effect is the same: making the code more polymorphic
reduces the space of possible implementations.

## Summary

Monomorphic type signatures are very concrete. They permit richly descriptive
variable names, because you know exactly the types you are working with and
what they semantically represent.

The dark side of monomorphic type signatures is that they have far more
inhabitants. There are so many ways to implement them, your chances of screwing
them up is comparatively high, and it's harder to reason about their correctness.

Introducing polymorphism can constrain the space of possible implementations and
make it simpler to mentally verify correctness of a piece of code. Completely
polymorphic type signatures don't permit descriptive variable names, but they
do vastly constrain the space of implementations.

Sometimes we can push the polymorphism very high in the code, and other times
we can't. But even when we need monomorphism, we can push as much functionality
as we can into little bubbles of polymorphism.

This lets us apply the technique virtually anywhere, and reduce the total amount
of functionality that is implemented monomorphically.

As with all refactoring techniques, it's possible to over-apply this technique,
as I'm sure you can imagine. But in general, I recommend experienced developers
look for *more* ways to polymorphize their code (not fewer).

At least for me, the resulting improvements in code quality have been quite
noticeable. If you've tried the technique, please share your experiences below!
