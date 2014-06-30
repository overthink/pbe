---
layout: post
title: sneakyThrow and throwing checked exceptions as unchecked
published: false
---

I recently came across some [crazy code in Square's
Okio](https://github.com/square/okio/blob/c200cb65ddf37fc4d11c01205ae1dd1eaf5f1136/okio/src/main/java/okio/Util.java#L64).
Here it is for easy reference:

{% highlight java %}
final class Util {
  /**
   * Throws {@code t}, even if the declared throws clause doesn't permit it.
   * This is a terrible – but terribly convenient – hack that makes it easy to
   * catch and rethrow exceptions after cleanup. See Java Puzzlers #43.
   */
  public static void sneakyRethrow(Throwable t) {
    Util.<Error>sneakyThrow2(t);
  }

  @SuppressWarnings("unchecked")
  private static <T extends Throwable> void sneakyThrow2(Throwable t) throws T {
    throw (T) t;
  }
}
{% endhighlight %}

Their comment admits it's nuts, and points to [Java
Puzzlers](http://www.amazon.com/Java%C2%BF-Puzzlers-Traps-Pitfalls-Corner/dp/032133678X),
which I wasn't aware of.  Apparently I'm out of it since this has been around
since 2005, and the technique is used in many open source libraries (search
around).

I had two questions: 1) how does this work, and 2) why would you ever do this?
Checked exceptions are a huge pain, but outright circumvention seems extreme.

After some study, the idea behind this code seems to be to make it possible to
throw any `Throwable` without declaring a `Throws` clause in the method
declaration.  Some exprimentation made it more clear.

{% highlight java %}
import java.io.IOException;

class Test {

  @SuppressWarnings("unchecked")
  public static <T extends Throwable> void sneakyRethrow2(Throwable t) throws T {
    throw (T) t;
  }

  public static void sneakyRethrow(Throwable t) {
    Test.<Error>sneakyRethrow2(t);
  }

  // Naively I wondered why this wasn't the same as what sneakyRethrow
  // ultimately does
  public static void doesntWork(Throwable t) throws Error {
    throw (Error) t;
  }

  public static void test1() {  // note: no Throws declaration!
    System.out.println("sneakyRethrow");
    sneakyRethrow(new IOException("foo"));
  }

  public static void test2() {  // note: no Throws declaration!
    System.out.println("doesntWork");
    doesntWork(new IOException("foo"));
  }

  public static void main(String[] args) {
    try {
      test1();
    } catch (Exception e) {
      System.out.println("Caught: " + e);
    }
    try {
      test2();
    } catch (Exception e) {
      System.out.println("Caught: " + e);
    }
  }
}
{% endhighlight %}

This compiles, but the `throw (Error) t;` line in `doesntWork` doesn't work for
most cases.  There's no guarantee you can cast an arbitrary `Throwable` to
`Error`.  Running the code demonstrates this:

  sneakyRethrow
  Caught: java.io.IOException: foo
  doesntWork
  Caught: java.lang.ClassCastException: java.io.IOException cannot be cast to java.lang.Error

Here is Java's exception class hierarchy, pinched from
[here](http://www.javamex.com/tutorials/exceptions/exceptions_hierarchy.shtml),
with '''unchecked''' exceptions in red:

![Java's exception hierarchy](/img/java-exception-hierarchy.png "Java's exception hierarchy")

At this point I still didn't know how this was possible.  I read the relevant
section of Java Puzzlers, but it didn't really explain the full details of the
hack, imo.  One important thing it did make me aware of was this: '''checked
exceptions are only part of Java, not the JVM'''.  They're only enforced by the
compiler.  In bytecode you can happily throw any exception from anywhere,
without restrictions.  Time to look at the bytecode.  Here are the relevant bits (`javap -c Test`):

    public static <T extends java/lang/Throwable> void sneakyRethrow2(java.lang.Throwable) throws T;
      Code:
         0: aload_0
         1: athrow

    public static void sneakyRethrow(java.lang.Throwable);
      Code:
         0: aload_0
         1: invokestatic  #2                  // Method sneakyRethrow2:(Ljava/lang/Throwable;)V
         4: return

    public static void doesntWork(java.lang.Throwable) throws java.lang.Error;
      Code:
         0: aload_0
         1: checkcast     #3                  // class java/lang/Error
         4: athrow

Looking at this bytecode it's obvious why `doesntWork` gets a
`ClassCastException` but `sneakyRethrow` doesn't.  There's no `checkcast`
operation in the `sneakyRethrow` code path.  How is this possible?
`sneakyThrow2` "clearly" casts the `Throwable` to `T` (which should be `Error`)
in this line:  `throw (T) t;`.  Then I clued in.  Another compiler-only feature
is being exploited here: generics.  At runtime all the generic type information
is "erased" -- it's not in the bytecode.  So `throw (T) t;` effectively becomes `throw t;`, and the
resulting bytecode shows this.  Contrast this with `doesntWork` where I've
explicitly cast the `Throwable` to `Error`.  Nothing is erased and we get bytecode with the
`checkcast` that leads to our `ClassCastException`.

## tl;dr

* checked exceptions are only checked in the compiler, the JVM doesn't care, you can throw anything
* type parameters are compiler-only too, there is no trace of them in the bytecode
* you can use type parameters in `throws` clauses when declaraing methods
* combine all of these for a cool hack

