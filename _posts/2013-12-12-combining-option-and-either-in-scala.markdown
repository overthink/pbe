---
layout: post
title: Combining Option and Either in Scala
description: Using Either in for comprehensions in Scala.
---

My last post on [nested if-let in Clojure]({% post_url 2013-12-02-nested-if-let-in-clojure %})
exists in part because I have worked with Scala for many years, and in Scala
it's there's a very handy syntax for the "only act when a bunch of optional
things are defined" situation.

{% highlight scala %}
val x: Option[Int] = ...
val y: Option[Int] = ...
val z: Option[Int] = ...

val result =
  for {
    a <- x
    b <- y
    c <- z
  } yield a + b +c
{% endhighlight %}

Here, `result` will be `None` unless all of `x`, `y`, `z` are defined.  This is
a common idiom in Scala, and a nice one, IMO.

But what if, like in the other post, we want to know which of the optional
things is undefined when something goes wrong?  (Constraint: without going
outside the standard library, i.e. no scalaz.)

Scala has an `Either` type which seems appropriate.  I assumed I could
"convert" my `Option`s to `Either`s and use the for comprehension syntax as
usual.  It generally works that way, but there's a twist that took me a bunch
of time to figure out.  Here's where I landed.

{% highlight scala %}
val x: Option[Int] = Some(1)
val y: Option[Int] = Some(2)
val z: Option[Int] = None

// Just look away from that return type.
def toRight[Ok, Err](x: Option[Ok], orElse: => Err): Either.RightProjection[Err, Ok] = {
  val either =
    x match {
      case Some(x) => Right(x)
      case None => Left(orElse)
    }
  either.right // WTF?  See below.
}

val result: Either[String, Int] =
  for {
    a <- toRight(x, { println("side effect!"); "x was None" })
    b <- toRight(y, "y was None")
    c <- toRight(z, "z was None")
  } yield {
    a + b + c
  }

result match {
  case Right(x) => println("Result was %d".format(x))
  case Left(msg) => println("Error, msg was '%s'".format(msg))
}

// prints "Error, msg was 'z was None'" in this case.
{% endhighlight %}

Confusingly to me, after converting the `Option` to a `Right` or `Left` you
have to project it to a RightProjection (chosing Right vs Left is simply
convention) since `Either` itself doesn't have a `flatMap` method.  I found this
especially bizarre because you are explicitly calling `.right` on the `Left` you
just created.  Types!  Anyway, I am not alone in finding this weird,
apparently.  Gory details and one fix proposal
[here](http://robsscala.blogspot.ca/2012/06/fixing-scalaeither-unbiased-vs-biased.html).

In fairness, this bit of localized weirdness results in fairly concise syntax
in the for comprehension itself.

