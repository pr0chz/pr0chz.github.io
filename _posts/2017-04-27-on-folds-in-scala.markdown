---
layout: post
title:  "On Folds in Scala"
date:   2017-04-27 09:17:43 +0200
categories: fold scala
---

Folds are one the basic functional building blocks for expressing operations on the sequences of data. There are
lot of other functions that can be expressed in terms of folds. But folds themselves has some interesting properties
which I want to explore in this post.

## What is a fold

Informally fold is an operation on an sequence which allows us to go through all items in that sequence while accumulating
some kind of result. It has two parameters - initial state of accumulator and a binary operation which incorporates a
single item from sequence together with previous accumulator state.

There are two types of folds - left fold and right fold. They differ in the direction they are traversing the sequence
and also some other things which we will explore here.

To give you an idea, here are their respective signatures for the `List` type:

{% highlight scala %}
def foldLeft[A, B](l: List[A])(z: B)(f: (B, A) => B): B
def foldRight[A, B](l: List[A])(z: B)(f: (A, B) => B): B
{% endhighlight %}

You can see that signatures are really differing only in the order of parameters of `f`, which is a merely a convention
to suggest in which order fold consumes the sequence.

## Left fold

Left fold consumes sequence from the first item, so the implementation for `List` might look like this:

{% highlight scala %}
def foldLeft[A, B](l: List[A])(z: B)(f: (B, A) => B): B = l match {
  case x :: xs => foldLeft(xs)(f(z, x))(f)
  case Nil => z
}
{% endhighlight %}

We'll better show how it works with some example:

{% highlight scala %}
foldLeft(List(1, 2, 3))(0)(_ + _)
{% endhighlight %}

We can unwind the evaluation of this expression like this:
```
foldLeft(List(1, 2, 3))(0)
foldLeft(List(2, 3))(0 + 1)
foldLeft(List(3))((0 + 1) + 2)
foldLeft(List())(((0 + 1) + 2) + 3)
((0 + 1) + 2) + 3
```

Now there are some interesting observations here. Implementation of `foldLeft` is clearly tail recursive. This means that
it is stack safe and will not fail even for very long `List`.

Second observation is that there are some use cases where you would like to prematurely finish the sequence traversal,
but we will fail to implement this with `foldLeft`.

Consider for example implementation of find:

{% highlight scala %}
def find[A](l: List[A])(p: A => Boolean):Option[A] =
  foldLeft(l)(None:Option[A])((acc, x) => acc.orElse(if (p(x)) Some(x) else None))
{% endhighlight %}

This searches for the first occurence of item which satisfies some predicate. So theoretically we should be able to
terminate the computation as soon as we have found some occurence. However have a look on `foldLeft` function -
there is really no way a function `f` can decide to not evaluate the rest of the `List`. We will always end up calling
`leftFold` recursively and thus traversing the whole sequence.

## Right fold

Now let's have a look at `rightFold`. This fold consumes sequence from the last item and for `List` its implementation
might go like this:

{% highlight scala %}
def foldRight[A, B](l: List[A])(z: B)(f: (A, B) => B): B = l match {
  case x :: xs => f(x, foldRight(xs)(z)(f))
  case Nil => z
}
{% endhighlight %}

Reusing the previous example:

{% highlight scala %}
foldRight(List(1, 2, 3))(0)(_ + _)
{% endhighlight %}

Evaluation will progress like this:

{% highlight scala %}
foldRight(List(1, 2, 3))(0)
1 + foldRight(List(2, 3))(0)
1 + (2 + (foldRight(List(3))(0)))
1 + (2 + (3 + foldRight(List())(((0 + 1) + 2) + 3)))
1 + (2 + (3 + 0))
{% endhighlight %}

This `foldRight` implementation is not tail-recursive and is not therefore stack safe. But when you have a look at
the way function `f` is applied, there might be a way how to prematurely stop the evaluation. So let's try and
make the second parameter lazily evaluated:

{% highlight scala %}
def foldRight[A, B](l: List[A])(z: => B)(f: (A, => B) => B): B = l match {
  case x :: xs => f(x, foldRight(xs)(z)(f))
  case Nil => z
}
{% endhighlight %}

This looks much better. If function `f` does not force the evaluation of the second parameter, recursion might stop
early. Let's try implementing the `find` function again, now with `foldRight`:

{% highlight scala %}
def find[A](l:List[A])(p: A => Boolean):Option[A] =
  foldRight(l)(None:Option[A])((x, acc) => (if (p(x)) Some(x) else None).orElse(acc))
{% endhighlight %}

We have to be careful to not use `acc` until needed. Since `orElse` method parameter is also lazily evaluated,
we stop the evaluation once we find the first item satisfying the predicate.

This is a great property of `foldRight`, but we have traded the early termination for the tail recursiveness.
Which is unfortunate, now we will be having stack problems with long lists.






