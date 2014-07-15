---
layout: post

title: sneakyThrow()&#58; Checked Exceptions as Unchecked
description: How sneakyThrow() is used to throw checked exceptions as unchecked in Java.
published: true
---

I recently came across some [suspicious code in Square's
Okio](https://github.com/square/okio/blob/c200cb65ddf37fc4d11c01205ae1dd1eaf5f1136/okio/src/main/java/okio/Util.java#L64):

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
which I wasn't aware of.  Apparently I'm out of touch since this has been around
since 2005, and the technique is used in many open source libraries (search
around).

I had two questions: 1) how does this work (!?), and 2) why would you ever do this?
Checked exceptions are a huge pain, but outright circumvention seems extreme.

Here's some code illustrating what `sneakyRethrow` allows one to do:

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

  // note: IOException is checked, but no Throws declaration
  public static void test1() {
    System.out.println("sneakyRethrow");
    sneakyRethrow(new IOException("foo"));
  }

  // note: IOException is checked, still no Throws declaration
  public static void test2() {
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
    System.out.println("done");
  }
}
{% endhighlight %}

This compiles, but the downcast `throw (Error) t;` line in `doesntWork` isn't
safe for most cases.  There's no guarantee you can cast an arbitrary
`Throwable` to `Error`.  Here is Java's exception class hierarchy, pinched from
[here](http://www.javamex.com/tutorials/exceptions/exceptions_hierarchy.shtml),
with __unchecked__ exceptions in red:

![Java's exception hierarchy](/img/java-exception-hierarchy.png "Java's exception hierarchy")

Running the code demonstrates the problem with the naive `doesntWork`, and also
shows `sneakyRethrow` doing its thing:

    sneakyRethrow
    Caught: java.io.IOException: foo
    doesntWork
    Caught: java.lang.ClassCastException: java.io.IOException cannot be cast to java.lang.Error
    done

At this point I still didn't know how this was possible.  I read the relevant
section of Java Puzzlers, but it didn't really explain the full details of the
hack, imo.  One important thing it did make me aware of was this: __checked
exceptions are only part of Java, not the JVM__.  They're only enforced by the
compiler.  In bytecode you can happily throw any exception from anywhere,
without restrictions.  Time to look at the bytecode (via `javap -c Test`):

{% highlight java %}
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
{% endhighlight %}

Looking at this bytecode it's obvious why `doesntWork` causes a
`ClassCastException` but `sneakyRethrow` doesn't: there's no `checkcast`
instruction in the `sneakyRethrow` code path.  How is this possible?
`sneakyThrow2` "clearly" casts the `Throwable` to `T` (which is `Error`) in
this line:  `throw (T) t;`.  Then I clued in.  Another compiler-only feature is
being exploited here: generics.  At runtime all the generic type information is
"erased" -- it's not in the bytecode.  So `throw (T) t;` effectively becomes
`throw t;` (no cast), and the resulting bytecode shows this.  Contrast this
with `doesntWork` where I've had to explicitly cast the `Throwable` to `Error`.
Nothing is erased and we get bytecode with the `checkcast` that leads to our
`ClassCastException`.

Now, when would be a good time to use `sneakyRethrow`?  Don't use it!  You
agreed to use Java, pay the tax and do the right thing with checked exceptions.
But then why has this code shown up in various respected Java projects?  The
one use case where `sneakyRethrow` seems to be accepted is when you catch
`Throwable` (i.e. the most generic type of exception), do a bit of cleanup,
then want to rethrow the `Throwable` as its most specific type.  It's well
explained in this [Android platform
code](https://android.googlesource.com/platform/libcore/+/jb-mr2-release/luni/src/main/java/libcore/util/SneakyThrow.java).
I'll reproduce here for posterity:

{% highlight java %}
// The following code must enumerate several types to rethrow:
public void close() throws IOException {
    Throwable thrown = null;
    ...
    if (thrown != null) {
        if (thrown instanceof IOException) {
            throw (IOException) thrown;
        } else if (thrown instanceof RuntimeException) {
            throw (RuntimeException) thrown;
        } else if (thrown instanceof Error) {
            throw (Error) thrown;
        } else {
            throw new AssertionError();
        }
    }
}

// With SneakyThrow, rethrowing is easier:
public void close() throws IOException {
    Throwable thrown = null;
    ...
    if (thrown != null) {
        SneakyThrow.sneakyThrow(thrown);
    }
}
{% endhighlight %}

So, like the comment in Okio says, using this in cleanup code is pretty
convenient.  Neat hack.

## Summary

* checked exceptions are only checked by the compiler, the JVM allows throwing any Throwable at any time
* type parameters are compiler-only too, there is no trace of them in the bytecode (erasure)
* combine these things to defeat the compiler and get the bytecode you want
* never use the resulting code
* ok, use it carefully in cleanup situations (e.g. after catching a Throwable)

