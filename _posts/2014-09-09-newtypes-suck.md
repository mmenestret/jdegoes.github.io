---
layout:       post
title:        "Newtypes Aren't As Cool As You Think"
description:  "Why newtyping that next data type may not buy you as much as you think it does."
category:     articles
tags:         [functional programming, ml, haskell, scala, idris, purescript]
---

My [last post](/articles/principled-typeclasses/) talked about what's wrong with type classes (in general, but also specifically in Haskell). This post generated some [great feedback on Reddit](http://www.reddit.com/r/haskell/comments/2dw3zq/haskells_type_classes_why_we_can_do_better/), including some valid criticism that I didn't explain why I hated on newtypes so much.

I took some of that feedback and incorporated it into a [revised version](/articles/principled-typeclasses/) of the post, but I have a *lot* more to say about "newtypes", so I decided to write another blog post.

## What's in a Newtype

Newtypes are a feature of Haskell that let you define a new type in terms of an existing type.

In the following example, I create a newtype for `Email`, which "holds" a `String`.

{% highlight haskell %}
newtype Email = Email String
{% endhighlight %}

They are similar to type synonyms (`type Email = String`), except that type synonyms don't create new types, they just allow you to refer to existing types by other names.

Every newtype can be easily translated into a data declaration. In fact, only the keyword changes:

{% highlight haskell %}
data Email = Email String
{% endhighlight %}

There's a slight semantic difference between the two, but for purposes of this blog post, any criticism I have against newtypes apply equally to similar constructs modeled with `type` or `data`.

## The Promise of Newtypes

The promise of newtypes is that we can cheaply make our code more type safe. Instead of passing around `String`s as emails, for example, we can create a super lightweight "wrapper" around a `String` called `Email`, and make it an error to use a `String` where an `Email` is expected.

Even in Java, it's considered good coding practice to wrap primitives with classes whose names denote the meaning of the wrapper (Email, SSN, Address, etc.).

There's part of this promise that's certainly true. If I have to define a function accepting 4 parameters, and 3 of them are strings, but one of those strings denotes an email, then I have two choices:

1. Model the email parameter with a `String`. In this case, I may accidentally use the email where I intended to use the other two string parameters, or I may use one of the other two string parameters where I intended to use the email. Considering just these choices, there are **five** ways my program may go wrong if I use the wrong name in the wrong position.
2. Model the email parameter with a `newtype`. In this case, I cannot use the email where I intended to use the other two string parameters, because the compiler may stop me. Similarly, I cannot use the other two string parameters where I intended to use the email, for the same reason. Looking at just these choices, there are **0** ways my program may go wrong.

Thus, newtypes, like all good FP practices, *reduce the number of ways my program can go wrong.*

Unfortunately, in my opinion, *they don't go nearly far enough.*

## False Security

For most intent and purposes, newtypes are *isomorphic* to the single value they hold.

In my preceding example, given a `String`, I can get an email (`Email "foo"`). Given an `Email`, I can also get a `String`, e.g. by pattern matching on the `Email` constructor.

Stated differently, and also approximately because I'm ignoring bottom: the `String` and `Email` types are isomorphic; they contain the same inhabitants, for any useful definition of "same". The only substantive difference between the preceding `String` and `Email` is the name of the data constructor (call `Email` and `AbergrackleFoozywatzit`, and what has changed?). 

Hence, my previous criticism of newtypes as "programming by name".

By themselves, newtypes don't *really* reduce the number of ways my program can go wrong. They just make it a bit harder to go wrong. But any newtype is isomorphic to the value it holds, and it's trivial to convert between the two.

In fact, if my code doesn't need to convert between the two (either directly or indirectly), then it's better off generic. That is, if I never need to convert an `Email` to a `String`, or a `String` to an `Email`, then I should really write the code generically to work with any value (even if that means making data structures or functions more polymorphic).

Parametrically provides a massive reduction in the number of ways my program can go wrong. Newtypes, on the other hand, just make it a bit harder to go wrong, by adding one layer of indirection.

In this example, as with many newtypes, I've created a bad isomorphism. The domain model of an email is not isomorphic to the data model of a string. But by using a newtype, I have implicitly declared that they *are* isomorphic.

Calling a string an email may make me *feel* better, because of the different name, but fundamentally, with a newtype, it's still a string, and I'm only ever *one more step away* from going wrong.

In my experience, a *lot* of newtypes create an isomorphism between things that, properly modeled, are *not* isomorphic. 

Fortunately, there's a well-worn *workaround* that lets us get more mileage out of newtypes.

## Smart Constructors

If I define `Email` in a module, I can make its data constructor private, and export a helper function to construct an `Email`. Such helper functions are called *smart constructors*.

They can be used to *break* the natural isomorphisms created by newtyping.

An example is shown below:

{% highlight haskell %}
newtype Email = Email String

mkEmail :: String -> Maybe Email
mkEmail s = ...
{% endhighlight %}

In this example, I create a smart constructor which does *not* promise that it can turn *every* string into an email. It promises only that it *might* be able to turn a string into an email, by returning a `Maybe Email`.

With the smart constructor approach, I've modeled the fact that while *every* email has a string representation, not *every* string is an email.

Going back to my earlier example of three parameters to a function, if I use a smart constructor, then while I can still use an email anywhere a string is expected (by converting), I can't use a string anywhere an email is expected. (Well, ignoring the `fromJust` abomination.)

### Smart Constructors, Dumb Data

Smart constructors take us one step closer toward modeling data in a type safe fashion. 

Unfortunately, I don't think it's far enough.

With smart constructors, our data model is fundamentally *underconstrained*, so we patch that up by restricting who can create the data. That's putting a band-aid on the real problem.

Why not just solve the root issue, that our data model is underconstrained?

## Dumb Constructors, Smart Data

The best solution to this problem, I believe, is creating a data model where there is a *true* isomorphism between the entity modeled by our data and the product of the values passed to the data constructor.

That is, there exists no regions in our data's state space which correspond to invalid states.

Email is a simple example, because there are [well-defined models](http://tools.ietf.org/html/rfc5322#section-3.4) for what constitutes a data model, which are straightforwardly if tediously translated into data declarations.

(To some extent, it's a failure of most languages I know that such specifications cannot be easily translated into code without any tedious boilerplate.)

When our data declaration precisely fits our data model specification, there's no need for smart constructors, and no need for newtypes. There's far fewer ways that code can go wrong, and because our domain model is captured precisely by our data model, we can transform that data model in ways that make semantic sense.

## Summary

As I've explained in this blog post, I don't *really* hate newtypes. I think they're very useful, and I *do* use them, because they make it more difficult for my programs to go wrong.

Ultimately, however, I think a lot of problems solved with newtypes (modeling coordinates, positions, emails, etc.) are better solved by more precise data modeling. By making our programs more honest about the fact that, for example, strings and emails are *not* isomorphic.

Precision may be tedious due to limitations of the language we work in, but even more tedious is debugging broken code.