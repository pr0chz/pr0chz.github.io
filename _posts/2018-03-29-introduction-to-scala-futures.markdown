---
layout: post
title:  "Introduction to Scala Futures"
date:   2018-03-29 09:17:43 +0200
categories: future scala
---

# Introduction to Scala Futures

## Basics - For Comprehension

`For` comprehensions are used throughout this article interchangeably with bare
`flatMap`, `map` and `filter` method calls. It is expected that reader is familiar 
with the concept and is able to understand how `for` is compiled into code.

Just a brief example:
{% highlight scala %}
for {
  x <- List(1, 2, 3) 
  y <- List(1, 2)
} yield x * y
{% endhighlight %}

Is a syntactic sugar for a sequence of `flatMap`s and `map`s:
{% highlight scala %}
List(1, 2, 3).flatMap { x =>
  List(2, 1).map(y => x * y)
}
{% endhighlight %}


## What is Future

`Future` is `Scala`'s construct for handling asynchronous operations. 

Asynchronicity, concurrency and multithreading are notoriously known
for being hard and error prone. Even though not being a silver bullet, 
`Future` gives us a nice framework to reason about async operations.

`Future[A]` is a placeholder for a value of type `A` which is being computed 
asynchronously. Result of the computation is handled when it becomes 
available using a callback (i.e. again asynchronously).

`Future` can also handle failures, in which case the completed `Future`
contains an `Exception` instead of regular result.

## Real World Example

Let's start with an example to give you a feel what are `Future`s about.

Most likely you will deal with `Future`s in some third party API - 
MongoDB, HTTP API etc. 

Imagine a `Future` based API for querying list of events associated
with some revision. API also contains method for getting the current
latest revision. Our task is to count the number of events in 
current list of events.

Don't worry if it is not clear how this works yet, we will return
to this code later:

{% highlight scala %}

import scala.concurrent.Future
import ExecutionContext.Implicits.global

case class Event(/* some content here */)

// async API - HTTP, Mongo etc.
trait Repo {
  def getCurrentRevision(): Future[Int]
  def getEvents(revision: Int): Future[List[Event]]
}

val repo: Repo = ...

// chained Futures and following computation
// note we still have a Future, not the actual result here

val eventCount: Future[Int] = for {
  revision <- repo.getCurrentRevision()
  events <- repo.getEvents(revision)
} yield events.size

{% endhighlight %}

## Creating Future

`Futures` are heavily used by libraries for IO so typically you just transform already
existing `Futures` and you do not often create your own. But for testing and 
mock objects creating your own `Future`s might be very useful.

Spawn a new computation:
{% highlight scala %}
val f: Future[Int] = Future(5 * 10)
{% endhighlight %}

This immediately creates and submits the task for processing. 

Or you can create an already completed Future:

{% highlight scala %}
val f1: Future[Int] = Future.successful(5)
val f2: Future[Int] = Future.failed(new Exception("Some failure"))
{% endhighlight %}

There is no computation done on these `Future`s, they already have a value.

## Execution Context

So how does the asynchronicity work? Let's inspect the method signatures on the 
actual `Future`:

{% highlight scala %}
trait Future[+T] extends Awaitable[T] {
  def onComplete[U](f: (Try[T]) ⇒ U)(implicit executor: ExecutionContext): Unit
}
{% endhighlight %}

`ExecutionContext` is an implicit parameter to all methods of `Future` which
spawn an async computation. Typically but not necessarily `ExecutionContext`
is a thread pool (backed by `ExecutorService` or `ForkJoinPool`) and `Future`
uses it to submit tasks for async execution.

There is a default `ExecutionContext` which defaults to `ForkJoinPool` and
the number of CPUs size. You can import it into scope like this:

{% highlight scala %}
import ExecutionContext.Implicits.global
{% endhighlight %}

There are some considerations about this default value which we will address later.

## Basic Callbacks

So how do we respond to the value computed by `Future`?

{% highlight scala %}
val f = Future(5 * 10)

// handle both at once - successful and failed Future execution
// onComplete converts Future[T] to Try[T]
f.onComplete {
  case Success(x) => println(s"Result is $x")
  case Failure(e) => e.printStackTrace()
}

// handle just successful
f.foreach(x => println(s"Result is $x"))

// handle failed Future
f.failed.foreach(e => e.printStackTrace())

// onSuccess and onFailure deprecated since 2.12
{% endhighlight %}

Note that all callbacks are also performed asynchronously so your main thread will
easily reach end of this code before anything is printed out. You can register
as many callbacks as you need, they will be executed in random order.

These callbacks are nice for just retrieving the result. But if we would like to chain
some other computation on the previous result, code will quickly become very ugly.

{% highlight scala %}
// chain another computation
f.foreach(x => {
  val f2 = Future(x * 5)
  f2.foreach(y => println(y))
})
{% endhighlight %}

## Higher level callbacks

`Future` also contains more high-level callbacks that we are used to use on 
collections:
* `map` - map value of `Future` to another value, just successful values are mapped
* `flatMap` - chain `Future`s into the sequence, just successful values are passed along
* `filter` - if condition not satisfied, fail the `Future`

We can now return to the original example. We need to chain two `Future`s
together as the second one needs the result of the first one, so we should
use a `flatMap` here.

{% highlight scala %}

case class Event(/* some content here */)

// async API - HTTP, Mongo etc.
trait Repo {
  def getCurrentRevision(): Future[Int]
  def getEvents(revision: Int): Future[List[Event]]
}

val repo: Repo = ...

// chained Futures

val eventCount: Future[Int] = repo.getCurrentRevision().flatMap(
  revision => repo.getEvents(revision).map(_.size)
) 

{% endhighlight %}

The last expression directly translates to code with `for` comprehension:
 
{% highlight scala %}
val eventCount: Future[Int] = for {
  revision <- repo.getCurrentRevision()
  events <- repo.getEvents(revision)
} yield eventSet.size
{% endhighlight %}

Which is more comprehensible and clear.

## Parallel vs sequential execution

`Future`s are by nature computed asynchronously, so there is no inherent
order in which they are executed. These two `Future`s will run concurrently
or in any arbitrary order:

{% highlight scala %}
val f1 = Future(5 * 3)
val f2 = Future("aaa")
{% endhighlight %}

For imposing order to processing, we usually use `flatMap` method, but in this
case this will not help you:

{% highlight scala %}
val x: Future[(Int, String)] = for {
  x1 <- f1
  x2 <- f2
} yield (x1, x2)
{% endhighlight %}

`f1` and `f2` will still run concurrently, just the processing of its results
will be chained.

If you want to impose an order to `Future` processing, you have to invoke
the `Future` operation from inside the `flatMap`:

{% highlight scala %}
val x: Future[(Int, String)] = for {
  x1 <- Future(5 * 3)
  x2 <- Future("aaa") // this Future is created after the first one finishes
} yield (x1, x2)
{% endhighlight %}

The second `Future` is created inside the lambda given to `flatMap` and thus is
executed in sequence after the first one.
 
Even in the case `Future`s are independent or returning `Unit`, you can chain them like this:

{% highlight scala %}
for {
  x1 <- computeInt() 
  _ <- saveIntToDb(x1)
} yield ()
{% endhighlight %}


## Error handling

Until now we have completely ignored that `Future` can fail. In the case of failure
`Future` will contain an `Exception` and for the most combinators the processing
will be just skipped propagating the error. Usually this behavior is ok, but sometimes
we need to recover from error - give default, try again etc.

{% highlight scala %}
val f: Future[Int] = Future {
  throw new Exception("Failed") // unhandled exception fails the Future
}

f recover { case e: Exception => 5 } // recover to successful Future with value 5
{% endhighlight %}

Other methods useful for error handling might be `recoverWith`, `fallbackTo`, `transform` 
and `transformWith`.

 
## If / filter behavior

A bit of suprising behavior comes with the `filter` usage:

{% highlight scala %}
val failedFuture = Future.successful(5).filter(_ == 6).map(_ * 2)

// or equivalently
for {
  x <- Future.successful(5)
  if x == 6
} yield x * 2
{% endhighlight %}

Intuitively this looks just like the value will be "left out" as in the case of collections.
But `filter` has to behave differently on `Future` - result will be **failed** `Future` with 
`NoSuchElementException`.
 
## Testing

You can use `Await` to wait for the result of `Future`. This is usually discouraged
in normal code, but perfectly acceptable in tests.

{% highlight scala %}
Await.ready(future, 1.seconds) // wait until completed 
Await.result(future, 1.seconds) // wait until completed and return result (or rethrow Exception)
{% endhighlight %}

`Await` will block your current thread until the `Future` is finished. This is dangerous 
behavior - **never use this in production code** (unless you know what you are doing).

`ScalaTest` has a nice support allowing you to return `Future[Assert]` from test function.
This basically removes the burden of waiting for the result, just map your `Future` to
`Assert` as a last step:

{% highlight scala %}
class Test extends AsyncFlatSpec {
  "5 * 3" should "be 15, even in Future" in {
    val f = Future(5 * 3)
    f map { x => assert(x === 15) }
  }
}
{% endhighlight %}

More info on [ScalaTest website](http://www.scalatest.org/user_guide/async_testing)

## Execution Context Gotchas

In more complex applications you need to be careful about thread management as it can easily 
become a mess. Different libraries / components are using different abstractions or 
approaches to managing concurrency and threads.

If you decide to use default `ExecutionContext` for `Futures` note that it is also 
used as default for parallel collections which can easily starve other tasks during
some longer computations. If you use parallel collection, you will need to create
your own `ExecutionContext` to avoid problems. `Akka` seems to be using different approach.

Also `ForkJoinPool` used as default backend is not recommended for long running tasks.
If you are blocking for longer time you should consider using 
[blocking construct](https://docs.scala-lang.org/overviews/core/futures.html#blocking-inside-a-future).

## Conclusion

We have shown a basic usage of `Future`s in Scala. 

Some more advanced topics were left out:
* Futures can sometimes become messy when you combine them with other `Monad` types 
(`Option`, `List`, ...). This boilerplate code can be removed by using monad transformers.
* Combining `Future`s with `Actor` model, which is yet another approach for asychronous
processing.





 
