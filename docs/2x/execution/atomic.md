---
layout: docs
title: Atomic
type_api: monix.execution.atomic.Atomic
type_source: monix-execution/jvm/src/main/scala/monix/execution/atomic/Atomic.scala
description: |
  References that can be updated atomically, for lock-free thread-safe programming, resembling Java's AtomicReference, but better.
---

Scala is awesome at handling concurrency and parallelism, providing
high-level tools for handling it, however sometimes you need to go
lower level. Java's library provides all the multi-threading
primitives required, however the interfaces of these primitives
sometime leave something to be desired.

One such example are the atomic references provided in
[java.util.concurrent.atomic](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/package-summary.html){:target="_blank"}
package. This project is an attempt at improving these types for daily
usage.

## Providing a Common interface

So you have
[j.u.c.a.AtomicReference&lt;V&gt;](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicReference.html){:target="_blank"},
[j.u.c.a.AtomicInteger](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicInteger.html){:target="_blank"},
[j.u.c.a.AtomicLong](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicLong.html){:target="_blank"}
and
[j.u.c.a.AtomicBoolean](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/atomic/AtomicLong.html){:target="_blank"}.
The reason why `AtomicReference<V>` does not suffice is because
compare-and-set works with reference equality, not structural equality
like it happens with primitives. So you cannot simply box an integer
and use it safely, plus you've got the whole boxing/unboxing overhead.

One problem is that all of these classes do not share a common
interface and there's no reason for why they shouldn't.

```scala
import monix.execution.atomic._

val refInt1: Atomic[Int] = Atomic(0)
val refInt2: AtomicInt = Atomic(0)

val refLong1: Atomic[Long] = Atomic(0L)
val refLong2: AtomicLong = Atomic(0L)

val refString1: Atomic[String] = Atomic("hello")
val refString2: AtomicAny[String] = Atomic("hello")
```

### Working with Numbers

One really common use-case for atomic references are for numbers to
which you need to add or subtract. To this purpose
`j.u.c.a.AtomicInteger` and `j.u.c.a.AtomicLong` have an
`incrementAndGet` helper. However Ints and Longs aren't the only types
you normally need. How about `Float` and `Double` and `Short`? How about
`BigDecimal` and `BigInt`?

In Scala, thanks to the
[Numeric[T]](http://www.scala-lang.org/api/current/index.html#scala.math.Numeric)
type-class, we can do this:

```scala
val ref = Atomic(BigInt(1))
// ref: monix.execution.atomic.AtomicNumberAny[scala.math.BigInt] = monix.execution.atomic.AtomicNumberAny@248a1241

// now we can increment a BigInt
ref.incrementAndGet()
// res4: scala.math.BigInt = 2

// or adding to it another value
ref.addAndGet(BigInt("329084291234234"))
// res6: scala.math.BigInt = 329084291234236
```

But then if we have a type that isn't a number:

```scala
val string = Atomic("hello")
```

Trying to apply numeric operations will of course fail:

```scala
string.incrementAndGet()
<console>:16: error: value incrementAndGet is not a member of monix.execution.atomic.AtomicAny[String]
       string.incrementAndGet()
              ^
```

### Support for Other Primitives (Float, Double, Short, Char, Byte)

Here's a common gotcha with Java's `AtomicReference<V>`. Suppose
we've got this atomic:

```scala
import java.util.concurrent.atomic.AtomicReference
// import java.util.concurrent.atomic.AtomicReference

val ref = new AtomicReference(0.0)
// ref: java.util.concurrent.atomic.AtomicReference[Double] = 0.0
```

The unexpected happens on `compareAndSet`:

```scala
val isSuccess = ref.compareAndSet(0.0, 100.0)
// isSuccess: Boolean = false
```

Calling `compareAndSet` fails because when using `AtomicReference<V>`
the equality comparison is done by reference and it doesn't work for
primitives because the process of
[Autoboxing/Unboxing](http://docs.oracle.com/javase/tutorial/java/data/autoboxing.html)
is involved. And then there's the efficiency issue. By using an
AtomicReference, you'll end up with extra boxing/unboxing going on.

`Float` can be stored inside an `AtomicInteger` by using Java's
`Float.floatToIntBits` and `Float.intBitstoFloat`. `Double` can be
stored inside an `AtomicLong` by using Java's
`Double.doubleToLongBits` and `Double.longBitsToDouble`. `Char`,
`Byte` and `Short` can be stored inside an `AtomicInteger` as well,
with special care to handle overflows correctly. All this is done to avoid boxing
for performance reasons.

```scala
val ref = Atomic(0.0)
// ref: monix.execution.atomic.AtomicDouble = monix.execution.atomic.AtomicDouble@38b899fb

ref.compareAndSet(0.0, 100.0)
// res9: Boolean = true

ref.incrementAndGet()
// res10: Double = 101.0

val ref = Atomic('a')
// ref: monix.execution.atomic.AtomicChar = monix.execution.atomic.AtomicChar@57e4b34

ref.incrementAndGet()
// res11: Char = b

ref.incrementAndGet()
// res12: Char = c
```

### Common Pattern: Loops for Transforming the Value

`incrementAndGet` represents just one use-case of a simple and more
general pattern. To push items in a queue for example, one would
normally do something like this in Java:

```scala
import collection.immutable.Queue
import java.util.concurrent.atomic.AtomicReference

def pushElementAndGet[T <: AnyRef, U <: T](ref: AtomicReference[Queue[T]], elem: U): Queue[T] = {
  var continue = true
  var update = null

  while (continue) {
    var current: Queue[T] = ref.get()
    var update = current.enqueue(elem)
    continue = !ref.compareAndSet(current, update)
  }
  update
}
```

This is such a common pattern. Taking a page from the wonderful
[ScalaSTM](https://nbronson.github.io/scala-stm/){:target="_blank"}, 
with `Atomic` you can simply do this:

```scala
val ref = Atomic(Queue.empty[String])
// ref: monix.execution.atomic.AtomicAny[scala.collection.immutable.Queue[String]] = monix.execution.atomic.AtomicAny@3471144a

// Transforms the value and returns the update
ref.transformAndGet(_.enqueue("hello"))
// res15: scala.collection.immutable.Queue[String] = Queue(hello)

// Transforms the value and returns the current one
ref.getAndTransform(_.enqueue("world"))
// res17: scala.collection.immutable.Queue[String] = Queue(hello)

// We can be specific about what we want extracted as a result
ref.transformAndExtract { current =>
  val (result, update) = current.dequeue
  (result, update)
}
// res19: String = hello

// Or the shortcut, because it looks so good
ref.transformAndExtract(_.dequeue)
// res21: String = world
```

Voilà, you now have a concurrent, thread-safe and non-blocking
Queue. You can do this for whatever persistent data-structure you
want.

NOTE: the transform methods are implemented using Scala macros, so
you get zero overhead by using them.

## Scala.js support for targeting Javascript

These atomic references are also cross-compiled to [Scala.js](http://www.scala-js.org/)
for targeting Javascript engines, because:

- it's a useful way of boxing mutable variables, in case you need to box
- it's a building block for doing synchronization, so useful for code that you want cross-compiled
- because mutability doesn't take *time* into account and `compareAndSet` does, atomic references and
  `compareAndSet` in particular is also useful in a non-multi-threaded / asynchronous environment

## Efficiency

Atomic references are low-level primitives for concurrency and because
of that any extra overhead is unacceptable.

### Boxing / Unboxing

Working with a common `Atomic[T]` interface implies boxing/unboxing of
primitives. This is why the constructor for atomic references always
returns the most specialized version, as to avoid boxing and unboxing:

```scala
val ref = Atomic(1)
// ref: monix.execution.atomic.AtomicInt = AtomicInt(1)

val ref = Atomic(1L)
// ref: monix.execution.atomic.AtomicLong = AtomicLong(1)

val ref = Atomic(true)
// ref: monix.execution.atomic.AtomicBoolean = AtomicBoolean(true)

val ref = Atomic("")
// ref: monix.execution.atomic.AtomicAny[String] = monix.execution.atomic.AtomicAny@2c91fd2c
```

Increments/decrements are done by going through the
[Numeric[T]](http://www.scala-lang.org/api/current/index.html#scala.math.Numeric)
provided implicit, but only for `AnyRef` types, such as `BigInt` and
`BigDecimal`. For Scala's primitives the logic has been optimized to
bypass `Numeric[T]`.

### Cache-padded versions for avoiding the false sharing problem

In order to reduce cache contention, cache-padded versions for all Atomic
classes are provided. For reference on what that means, see:

- http://mail.openjdk.java.net/pipermail/hotspot-dev/2012-November/007309.html
- http://openjdk.java.net/jeps/142

To use the cache-padded versions, you need to override the default
`PaddingStrategy`:

```scala
import monix.execution.atomic.PaddingStrategy.{Left64, LeftRight256}

// Applies padding to the left of the value for a cache line of 64 bytes
val ref = Atomic.withPadding(1, Left64)

// Applies padding both to the left and the right of the value for
// a total object size of at least 256 bytes
val ref = Atomic.withPadding(1, LeftRight256)
```

The strategies available are:

- `NoPadding`: doesn't apply any padding, the default
- `Left64`: applies padding to the left of the value, for a cache line of 64 bytes
- `Right64`: applies padding to the right of the value, for a cache line of 64 bytes
- `LeftRight128`: applies padding to both the left and the right, for a cache line of 128 bytes
- `Left128`: applies padding to the left of the value, for a cache line of 128 bytes
- `Right128`: applies padding to the right of the value, for a cache line of 128 bytes
- `LeftRight256`: applies padding to both the left and the right, for a cache line of 256 bytes

And now you can join the folks that have mechanical sympathy :-P

