---
layout: post
title:  "Introduction to Scala Futures"
date:   2018-04-08 12:00:00 +0200
categories: future scala
---

## Basic terms

Definition of terms as they will be used in this article:

* **Concurrency** - Dealing with multiple in-progress things at once.
* **Paralellism** - Actually doing multiple things at once (on more CPU cores).

Note that concurrency does not imply parallelism. Concurrency can be achieved on a single CPU core by time-slicing 
individual concurrent tasks.

* **Synchronous task** - The task is executed after the previous one has ended.
* **Asynchronous task** - The task is executed concurrently with other tasks. This usually means that the task is 
executed somewhere else than "here", so that we do not care about result until later and continue with our 
operations.

Again asynchronous tasks do not imply anything about paralellism.

* **Thread** - A separate line of execution. Abstraction used by OSes to support concurrency. A thread is usually 
contained within a process. There can be multiple threads in a single process, sharing memory and executing 
concurrently.
* **Multithreading** - Using multiple threads in a single application.

## So what is a Future

In a low-level concurrent application you have to manage threads yourself and implement synchronization between
them (locks, synchronized queues, atomics etc). This is usually a very complex task.

* `Future` is Scala's approach to handle concurrency
* A placeholder for *asynchronously* computed value that will be available in the future. 
* Allows you to react *synchronously* to computed value with callbacks.
* Each `Future` can end up successfully completed or failed. In the latter case it will contain an `Exception`.

## Task in a Future

How to run a piece of code in the `Future`?

{% highlight scala %}
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

val f: Future[Int] = Future(5 * 10) // or Future { ... some larger code ... }
{% endhighlight %}

This example shows how to create a new `Future` task. `5 * 10` expression will be computed *asynchronously*. Result
of the computation is available only via `Future` API and we will have to wait until the `Future` is completed if we 
want to process it.

But how does this piece of code manage to be *asynchronous*? Some kind of magic? Notice `ExecutionContext` in the 
imports. This line imports default `ExecutionContext` which is backed by `ForkJoinPool` with size equal to number of 
CPU cores. You can imagine `ExecutionContext` as a thread pool where you can submit concurrently running tasks. All 
methods on `Future` which are creating a new `Future` or create a callback accept an implicit parameter of 
`ExecutionContext`:

{% highlight scala %}
object Future {
  def apply[T](body: ⇒ T)(implicit executor: ExecutionContext): Future[T]
}
{% endhighlight %}

`Future.apply` is actually side-effecting and submits the computation for processing into `ExecutionContext`. `Future`
API gives you the *concurrency*, `ExecutionContext` then controls the *parallelism* - no problem in giving it a 
single thread.

If you want to create an already completed `Future`, you can use `successful` and `failed` functions:

{% highlight scala %}
val f1: Future[Int] = Future.successful(5)
val f2: Future[Int] = Future.failed(new Exception("Some failure"))
{% endhighlight %}

There is no computation done on these `Future`s, they already have a value and do not need `ExecutionContext`.
Beware of putting some more complicated computations into these methods as they will be executed on caller's thread.

## Processing a Future's result

As we already mentioned `Future` results are processed via callbacks.

Let's consider this very simple API:

{% highlight scala %}
case class Event(/* some content here */)

// async API - HTTP, Mongo etc.
trait Repo {
  def getCurrentRevision(): Future[Int]              // current revision number                                                   
  def getEvents(revision: Int): Future[List[Event]]  // returns list of events associated with revision
}
{% endhighlight %}

Now, say that we want to get the current revision:

{% highlight scala %}
val revisionFuture = repo.getCurrentRevision()

// converts the result to Try[Int] and passes it to our lambda
revisionFuture.onComplete {
  case Success(revision) => println(s"Revision is $revision")
  case Failure(e) => e.printStackTrace()
}
{% endhighlight %}

`onComplete` registers a given lambda function as a callback and runs it after `revisionFuture` is completed. Note 
that your main thread might have already finished before this happens, there is no blocking involved.

Some more examples:

{% highlight scala %}
// handle just successful case
revisionFuture.foreach(revision => println(s"Revision is $revision"))

// handle failed Future, failed converts failed Future to Future[Exception]
revisionFuture.failed.foreach(e => e.printStackTrace())
{% endhighlight %}

## Chaining operations

Now, say that we want the actual list of events associated with the latest revision.

{% highlight scala %}
val revisionFuture = repo.getCurrentRevision()

// converts the result to Try[Int]
revisionFuture.onComplete {
  case Success(revision) => 
    val eventsFuture = repo.getEvents(revision)
    eventsFuture.onComplete {
      case Success(events) => println(s"Events are $events") 
      case Failure(e) => e.printStackTrace()
    }
  
  case Failure(e) => e.printStackTrace()
}
{% endhighlight %}

This is not very comprehensible. Moreover it would be beneficial if we could produce a value of
`Future[List[Event]]` which we can easily pass along. Error handling also looks cumbersome.

That is the point where `Future.flatMap` method comes into play:

{% highlight scala %}
class Future[T] {
  def flatMap[S](f: (T) ⇒ Future[S])(implicit executor: ExecutionContext): Future[S]
}
{% endhighlight %}

`flatMap` is useful for chaining and ordering `Future`s one after another, effectively making the operations
synchronous. Semantics is as follows:
* When there is a successful value, it is passed into the function and computation continues.
* When there is a failure, it is just propagated.

So let's try to chain our `Future`s together via `flatMap`:

{% highlight scala %}
val eventsFuture: Future[List[Event]] = repo.getCurrentRevision()
  .flatMap(revision => repo.getEvents(revision))

eventsFuture.onComplete {
  case Success(events) => println(s"Events are $events") 
  case Failure(e) => e.printStackTrace()
}
{% endhighlight %}

This looks much better. We have the `Future` of final type at the high-level and error is automatically 
propagated for us.

Now what happens if we are interested just in the number of events, not the full list. Let's use `map` function:

{% highlight scala %}
val eventsFuture: Future[Int] = repo.getCurrentRevision()
  .flatMap(revision => repo.getEvents(revision))
  .map(list => list.size)
{% endhighlight %}

`map` behaves similarly to `flatMap` regarding success and failure handling and behaves exactly as how you would 
expect - it maps a value contained in a `Future` to some other value of possibly other type:
  
{% highlight scala %}
class Future[T] {
  def map[S](f: (T) ⇒ S)(implicit executor: ExecutionContext): Future[S]
}
{% endhighlight %}  
  
Since we are using just `map` and `flatMap`, we can make our code even nicer by using `for` comprehension:

{% highlight scala %}
val eventsFuture: Future[Int] = for {
  revision <- repo.getCurrentRevision()
  events <- repo.getEvents(revision)
} yield events.size
{% endhighlight %}

So what exactly have we done by "chaining" our `Future`s? Whole computation now consists of three operations
and they are processed *synchronously* - one after another, but possibly each on a different thread. However 
this `Future` as a whole is running *asynchronously* and *concurrently* with regard to the main thread (and
possibly other `Future`s)

## Futures concurrency

In the last section we saw that `flatMap` is useful in sequencing the operations. But what if our operations
are independent and we would like them to run *concurrently*? And how to distinguish these two cases?

Generally a `Future` spawns its task on the place where it is created. In the case of an external API
(as we have) each of our call will produce new `Future`. The concurrency is then determined by the place
where we make that API call. You can usually identify a method that creates a new `Future` by its signature - 
it does not accept any `Future` parameter but it returns a `Future`.

Let's say we know two different revision numbers and we would like to compare their list of events.

This will make two API calls concurrently:

{% highlight scala %}
val rev1 = 1000
val rev2 = 1100

val eventsFuture1 = repo.getEvents(rev1)   // first Future is started
val eventsFuture2 = repo.getEvents(rev2)   // second Future is started
// running concurrently now

for {
  events1 <- eventsFuture1     // wait for first result
  events2 <- eventsFuture2     // wait for second result
} yield compare(events1, events2)
{% endhighlight %}

If you want to prevent this and make call synchronous even if they are not dependent on each other, you should
create the `Future` inside the `flatMap` function:

{% highlight scala %}
for {
  events1 <- repo.getEvents(rev1)   // first Future started, wait for the result
  events2 <- repo.getEvents(rev2)   // second Future started, wait for the result
} yield compare(events1, events2)                                                   
{% endhighlight %}

## A bit more complicated example

Let's create a slightly more complex version of our `Repo` interface:

{% highlight scala %}
trait Repo {
  def getCurrentRevision(): Future[Int]              // current revision number                                                   
  def getEventIds(revision: Int): Future[List[Int]]  // returns list of event ids associated with the revision
  def getEvent(id: Int): Future[Event]               // returns event for event id
  
  // implement this one
  def getEvents(revision: Int): Future[List[Event]]  // returns list of events associated with the revision
}
{% endhighlight %}

We would like to implement `getEvents` in terms of `getEventIds` and `getEvent`.

{% highlight scala %}
def getEvents(revision: Int): Future[List[Event]] = {
  
  def fetchEvents(ids: List[Int]): Future[List[Event]] = {
    val eventFutures: List[Future[Event]] = ids.map(getEvent)
    Future.sequence(eventFutures)
  }

  val result: Future[List[Event]] = for {
    ids <- getEventIds(revision)
    events <- fetchEvents(ids)
  } yield events
  
  result
}
{% endhighlight %}

Notice that we will spawn request for each of the event concurrently. This can easily work as a DoS attack for the
underlying service if there are too many event ids.

Alternatively we can use a fold to query events one by one:

{% highlight scala %}
def getEvents(revision: Int): Future[List[Event]] = {
  
  def fetchEvents(ids: List[Int]): Future[List[Event]] = {
    def addEvent(listFuture: Future[List[Event]], id: Int): Future[List[Event]] = for {
      list <- listFuture
      event <- getEvent(id)
    } yield event :: list

    ids.foldLeft(Future.successful[List[Event]](Nil))(addEvent)
  }
  
  val result: Future = for {
    ids <- getEventIds(revision)
    events <- fetchEvents(ids)
  } yield events
  
  result
}
{% endhighlight %}

## Random notes

### Multiple callbacks on a single Future

It is perfectly valid to process the value of single `Future` multiple times. The value is computed just once.

{% highlight scala %}
val f = Future(3 * 5)
f.foreach(println)
val f2 = f.map(_ * 2)
{% endhighlight %}

Both callbacks will be executed concurrently once `f` is completed (in no particular order).

### Error handling

It is very easy to forget that `Future` can fail. We should always consider the edge cases
and at least log the error as it may be easily (and silently) lost.

So how exactly does a `Future` fail? It catches an unhandled exception that happens in one of 
`Future`'s operation:

{% highlight scala %}
val f1 = Future { throw new Exception("Failed") }
val f2 = Future(1 + 2).map(_ => throw new Exception("Failed"))

f2 recover { case e: Exception => 5 } // recover to successful Future with value 5
{% endhighlight %}

Other methods useful for error handling might be `recoverWith`, `fallbackTo`, `transform` 
and `transformWith`.

### Futures and logging 

Quite often you would need to log a `Future` value that's buried in a long chain of transformations.
`foreach` does not return a value, so it is not very handy for this purpose.

In this case you should use `andThen` method. It accepts a function with `Try` parameter but does not influence
the returned `Future` in any way, even if your function throws an exception (that will be ignored). It is purely
for side-effecting purposes.

{% highlight scala %}
Future(1 + 2)
  .map(_ * 2)                         // transformation
  .andThen { case x => log.debug(x) } // Success(6)
  .map(_ * 2)                         // following transformation
{% endhighlight %}
 
### If / filter behavior

A bit of suprising behavior comes with the `filter` usage:

{% highlight scala %}
val failedFuture = Future.successful(5).filter(_ == 6).map(_ * 2)
{% endhighlight %}

Intuitively this looks just like the value will be "left out" as in the case of collections.
But `filter` has to behave differently on `Future` - result will be **failed** `Future` with 
`NoSuchElementException`.

Same holds for `if` statements inside `for` comprehensions.
 
### Waiting for the Future

You can use `Await` to wait for the result of `Future`.

{% highlight scala %}
Await.ready(future, 1.seconds)  // wait until completed 
Await.result(future, 1.seconds) // wait until completed and return result (or rethrow Exception)
{% endhighlight %}

`Await` will block your current thread until the `Future` is finished. This is dangerous 
behavior - never do this unless you know what you are doing.

One of the examples where this might be useful is a simple cmdline application relying on `Future` API - 
in that case it makes sense to block the main thread until the job is finished and then terminate the application.
 
### Testing

There are two approaches available in `ScalaTest` for testing `Future`s:

{% highlight scala %}
class Test extends FlatSpec with ScalaFutures {
  "5 * 3" should "be 15, even in Future" in {
    val result = Future(5 * 3)
    assert(result.futureValue == 15)
  }
}
{% endhighlight %}

More info in [ScalaTest docs](http://doc.scalatest.org/3.0.1-2.12/org/scalatest/concurrent/ScalaFutures.html)

Second approach is allowing you to return `Future[Assert]` from test function. Just map your `Future` to
`Assert` as a last step:

{% highlight scala %}
class Test extends AsyncFlatSpec {
  "5 * 3" should "be 15, even in Future" in {
    Future(5 * 3).map(x => assert(x === 15))
  }
}
{% endhighlight %}

More info on [ScalaTest website](http://www.scalatest.org/user_guide/async_testing)

### ExecutionContext gotchas

If you decide to use default `ExecutionContext` for `Futures` note that it is also 
used as default for parallel collections which can easily starve other tasks during
some longer computations. If you use parallel collection, you will need to create
your own `ExecutionContext` to avoid problems.

Also `ForkJoinPool` used as default backend is not recommended for long running tasks.
If you are blocking for longer time you should consider using 
[blocking construct](https://docs.scala-lang.org/overviews/core/futures.html#blocking-inside-a-future).






