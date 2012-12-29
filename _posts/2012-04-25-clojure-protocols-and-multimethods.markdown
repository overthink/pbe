---
layout: post
title: Clojure Protocols & Multimethods
---

This article is about Clojure's [multimethods](http://clojure.org/multimethods) and [protocols](http://clojure.org/protocols): what they are, how they work, and when you might want to use them.  The "when" part has always tripped me up -- especially with multimethods.  I'll look at some code from projects in the wild to help gain some intuition.

## Polymorphism

- polymorphic functions vs. polymorphic types clarification
- universal (single function body w/ type params or subtype poly) vs ad hoc polymorphism (multiple function bodies dispatched to)

Multimethods and protocols are ways to get [polymorphism in Clojure](http://blog.8thlight.com/myles-megyesi/2012/04/26/polymorphism-in-clojure.html).  What does this actually mean?  You can search around for the formal definitions, but I just mean that you want to write functions that behave differently based on their arguments.  Specifically:

1. You want a function `f` with a fixed signature (interface) that behaves differently depending on the types (or the values!) of its arguments.
1. There's a reasonable extensibility story so users of `f` can extend its behaviour without having its source code.

Why do you want this stuff?  So you can write your code in terms of abstractions rather than concrete implementations.  This is a familiar concept, especially if you come from a Java-ish OOP background.  In Java you get this sort of polymorphism via class or interface inheritence.

- example: Adder.add(a,b)
- implement Sum in terms of Adder.add

{% highlight clojure %}
(defn dumb-combine
  "Combine the args in some way depending on their types."
  [& args]
  (cond
    (every? coll? args) (apply concat args)
    (every? number? args) (apply + args))

(dumb-combine [1 2] ["a" "b"] [\c \d])
; (1 2 "a" "b" \c \d)
(dumb-combine 1 2 3 4 5)
; 15
(dumb-combine 1 "a")
; nil
{% endhighlight %}


{% highlight java %}

public interface Adder {
  public Object add(Object... args) {
  }
}

public class CollectionAdder implements Adder { ... }

public class NumberAdder implements Adder { ... }

public class App {
  public static Adder getBestAdder(Object... args) {
    // return a new Adder implementation that can deal with args
  }

  public static void main(String[] args) {
    Adder adder = getBestAdder(args);
    adder.add(args);
  }
}



public interface Adder {
  // Return true if this instance of Adder can handle args
  public Object canAdd(Object... args): Object;
  // Return args "added" together or null.  Implementor decides what add means.
  public Object add(Object... args): Object;
}

public class CollectionAdder implements Adder { ... }

public class NumberAdder implements Adder { ... }

class AdderFactory { 
  public void register(Adder: adder) {
    // keep a list of all registered adders
  }
  public Adder adderFor(Object... args) { 
    // return the first registered adder where canAdd(args) is true
  }
}

// client app (pseudo code)
factory = new AdderFactory();
factory.register(new CollectionAdder());
factory.register(new NumberAdder());
factory.adderFor(args).add(args);

{% endhighlight %}

This works ok: the `Adder` interface is public and anyone can create their own adder and register it with the appropriate AdderFactory.  

Java-style OOP is so popular and successful that polymorphism and inheritence are synonymous to many people (myself included at one point).

## References

* [Polymorphism in Clojure](http://blog.8thlight.com/myles-megyesi/2012/04/26/polymorphism-in-clojure.html) - Great article showing how polymorphism can be achieved with multimethods, protocol, and function values.
* http://lucacardelli.name/Papers/OnUnderstanding.A4.pdf section 1.3
* http://www.haskell.org/tutorial/classes.html
* http://stackoverflow.com/questions/6730126/parametric-polymorphism-vs-ad-hoc-polymorphism/6885445#6885445
* http://stackoverflow.com/questions/5854581/polymorphism-in-c/5854862#5854862 -- excellent!

------------------

## Intro

This article tries to explain Clojure [multimethods](http://clojure.org/multimethods) and [protocols](http://clojure.org/multimethods): what they are and when you might use them.

## Terminology

These aren't precise definitions, but just what I mean when I use these terms in this article.

Dispatch
: Select at runtime the appropriate code to execute for some named function.  Also known as [dynamic dispatch](http://en.wikipedia.org/wiki/Dynamic_dispatch).

Polymorphic function
: A single function `f` with several implementations.  The implementation that is actually executed depends on the arguments to `f`.  i.e. `f` dispatches to the appropriate implementation based on its arguments.

## Multimethods

A multimethod is a function with many implementations.  When a multimethod is defined, the user must provide a so-called "dispatching function" that is used to determine which implementation to execute for a given set of arguments to the multimethod.

Example.



## Protocols

## Similarities & Differences

same
- open for exetension

different
- protocol only dispatches on type of first arg
- protocols define a set of related functions, multimethod is just one function





