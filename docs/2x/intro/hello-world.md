---
layout: docs
title: Hello World!
description: "Go reactive (✿◠‿◠)"
---

Let's do instant gratification stuff.

First, we need a [Scheduler]({{ site.api2x }}#monix.execution.Scheduler)
whenever asynchronous execution happens.

```scala
// We need a scheduler whenever asynchronous
// execution happens, substituting your ExecutionContext
import monix.execution.Scheduler.Implicits.global

// Needed below
import scala.concurrent.duration._
```

## Task, the Lazy Future

For using [Task]({{ site.api2x }}#monix.eval.Task) or
[Coeval]({{ site.api2x }}#monix.eval.Coeval), usage is
fairly similar with Scala's own `Future`, except that
`Task` behaves lazily:

```scala
import monix.eval._

// A specification for evaluating a sum,
// nothing gets triggered at this point!
val task = Task { 1 + 1 }
```

`Task` in comparison with Scala's `Future` is lazy,
so nothing gets evaluated yet. We can convert it into
a `Future` and evaluate it:

```scala
// Actual execution, making use of the Scheduler in
// our scope, imported above
val future = task.runAsync
// future: monix.execution.CancelableFuture[Int]

// This is an evaluating Future, so we can use its result
future.foreach(x => println(s"Result: $x"))
// Result: 2
```

## Observable, the Lazy & Async Iterable

We already have the `Scheduler` in our local scope,
so lets make a tick that gets triggered every second.

```scala
import monix.reactive._

// Nothing happens here, as observable is lazily
// evaluated only when the subscription happens!
val tick = Observable.interval(1.second)
  // common filtering and mapping
  .filter(_ % 2 == 0)
  .map(_ * 2)
  // any respectable Scala type has flatMap, w00t!
  .flatMap(x => Observable.fromIterable(Seq(x,x)))
  // only take the first 5 elements, then stop
  .take(5)
  // to print the generated events to console
  .dump("Out")

// Execution happens here, after subscribe
val cancelable = tick.subscribe()
// 0: Out-->0
// 1: Out-->0
// 2: Out-->4
// 3: Out-->4
// 4: Out-->8
// 5: Out completed
```

Isn't this awesome? (✿ ♥‿♥)
