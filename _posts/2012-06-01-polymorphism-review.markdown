---
layout: post
title: All About Polymorphism
---

I recently became aware that I didn't know anything about polymorphism.  I
couldn't, for example, produce answers to these quesitons:

* What is ad-hoc polymorphism?
* If ad-hoc polymorphism is a thing, then what is "plain" polymorphism?
* What is a polymorphic function vs. a polymorphic type?
* What is compile-time polymorphism vs. runtime polymorphism?

To me, polymorphism was vaguely connected to implementing interfaces and
extending base classes -- i.e. I had no real understanding.  I searched around
and the result was a bunch of unsatisfying, complicated, and
sometimes-conflicting definitions of polymorphism.  I decided to dig a little
deeper and this article is the result.

## First: what's the point of polymorphism?

This is just my opinion, and there may be much better academic motivations, but
here's why I think it matters.  You can:

* write less code
* write more extensible code (handle types that don't exist yet)
* write simpler APIs (less for callers to remember)

## Poly Polymorphism

The first important thing I learned -- and it's obvious from my initial
questions -- is that there are many varieties of polymorphism.  Here's a nifty
chart lifted from [a great paper by Cardelli and Wegner][1] that shows the main
varieties and how they're related:

![Varieties of polymorphism](/img/varieties-of-polymorphism.png "Varieties of polymorphism")

Let's examine each node in this tree, starting at the top.

Polymorphism
: Values (including functions) and variables may have more than one type.

Yes, that's it.  The concept of polymorphism with no other context is very
general, so you don't get a very exciting definition.   Here's an example of a
polymorphic value in Java:

{% highlight java %}
  ArrayList<String> xs1 = new ArrayList<String>() { { add("A"); add("B"); add("C"); } };
  Collection<String> xs2 = xs1;
  List<String> xs3 = xs1;
{% endhighlight %}

This code doesn't do anything useful, but it's valid and compiles without
issue.  The `ArrayList` value referred to by `xs1` also has types
`Collection<String>` and `List<String>`, and can be successfully assigned to
variables of those types: `xs2` and `xs3`.  (Ingore for the moment that the
types are related in a special way -- we'll get to that in subtype polymorphism
below.)  

A problem with this particular definition of polymorphism is that it's probably
not what most developers are thinking about when they talk about polymorphism.
They probably mean something much more specific like _subtype polymorphism_
(we'll look at this below) or even _overloading_.  You have to carefully
examine the context to figure out what is really meant. This is probably a
source of much confusion when reading about polymorphism online (it was for me,
anyway).

Universal polymorphism
: Values can take on an infinite number of types that may share some common
structure.  e.g. in Java the "shared structure" might be specified as an
interface or base class.

"Infinite types" is a bit confusing.  I'm using the terminology from the
Cardelli paper.  My interpretation is that it means any number of future types
can be introduced, and as long as they have the required "shared sturcture", no
code in "universally polymorphic" functions would need to change.

Ad-hoc polymorphism
: Functions can take on a finite set of types that may have no structure at all
in common.

The implication here is that if you have an "ad-hoc polymorphic" function and
you want to extend it to work with some new type `T`, then you're going to have
to write some new code.  This is because -- by definition -- you can't assume anything
about the structure of `T`.

Now, the rest of the chart is specific enough that we can show real examples.
We'll go through each below.

### Subtype Polymorphism

Subtype polymorphism is a form of universal polymorphism, so values can take on
an infinite number of types.  But, in this case the allowed types must be
related in a special way: they must be in the same subtype hierarchy.
Paraphrasing Wikipedia, a [subtype][10] `S` is related to type `T` by some
concept of of _substitutability_.  That is, you can use type `S` anywhere that
type `T` is expected.  Repeatedly apply this relation and you get a hierarchy
of types.

> Subtype polymorphism is sometimes called inclusion polymorphism.  _Inclusion_
> in this context just means [_subset_][7].  A is an inclusion of B if A is a
> subset of B.  A bunch of nested subsets form a hierarchy.  You can see the
> parallel to subtyping as explained above.

Let's look at the canonical example:

{% highlight scala %}
trait Animal {
  def makeNoise()
}
class Cat extends Animal {
  def sitOnKeyboard() { ... }
  def makeNoise() { println("Meow") }
}
class Dog extends Animal {
  def fetchBone() { ... }
  def makeNoise() { println("Woof") }
}
object Example {
  def noiseMaker(a: Animal) { a.makeNoise() }
  def main(args: Array[String]) {
    val a1 = new Cat
    val a2 = new Dog
    noiseMaker(a1) // 'Meow'
    noiseMaker(a2) // 'Woof'
  }
}
{% endhighlight %}

In this example `Example#noiseMaker(a: Animal)` is a polymoprhic function because its parameter `a` accepts a value of type `Animal` for which any number of subtypes might exist.

TODO: get a non-stupid example from a popular Java library

### Parametric Polymorphism

In parametric polymorphism a function has one or more type parameters.  The parameter is bound at the time of function application.  Here's an example in Java:

{% highlight java %}
import java.util.List;
import java.util.ArrayList;

class Test {
  public static <T> T head(List<T> xs) {
    return xs.get(0);
  }
  public static void main(String[] args) {
    final List<String> xs = new ArrayList<String>() { { add("A"); add("B"); add("C"); } };
    final List<Integer> ys = new ArrayList<Integer>() { { add(1); add(2); add(3); } };
    System.out.println(head(xs)); // "A"
    System.out.println(head(ys)); // "1"
  }
}
{% endhighlight %}

The generic function `head` here has type parameter `T` and is an example of parametric polymorphism.  In this case the type doesn't matter at all since the function is written entirely in terms of the container type `List`.  You write the function `head` once, and reuse it with different types.

Here's another example of parametric polymorphism, this time in Haskell which is easier on the eyes:

{% highlight haskell %}
rev :: [a] -> [a]
rev [] = []
rev (x:xs) = rev xs ++ [x]

main :: IO ()
main = do putStrLn $ show $ rev ["A", "B", "C"] -- ["C","B","A"]
          putStrLn $ show $ rev [1, 2, 3]       -- [3,2,1]
{% endhighlight %}

Function `rev` is a generic function that will reverse any list.  This time the type parameter is `a`, and once again it doesn't matter what `a` ends up being when the function is applied.

### Overloading

Overloading is a form of ad-hoc polymorphism.  Previously, it never occurred to me that overloading had anything to do with polymorphism, but given the definitions above, it obviously does:

{% highlight scala %}
class Serializer {
  def toJson(x: String): String = "\"" + x + "\""  // yes, naive
  def toJson(x: Int): String = x.toString
}
{% endhighlight %}

Here we've provided two implementations of `toJson` that accept different, **unrelated** types as arguments (`String` and `Int`) and there's no shared code in the two implementations.  This is exactly our definition of ad-hoc polymorphism above.

This was a bit anti-climactic for me.  I'd primarily seen _ad-hoc_ used when talking about typeclasses and protocols, and I assumed there'd be more to the concept.  The key realization for me was that **typeclasses and protocols are really just structured overloading**. They allow you to add an additional implementation of some existing function, but with different argument types, and the compiler or runtime yells at you out if you try to use said function in the wrong way.

Continuing a bit with the JSON serializer example, let's say we need to serialize `Boolean`s as well as `String`s and `Int`s, but we don't have access to the source code of `Serializer`.  In an object oriented language we might do this:

{% highlight scala %}
class MySerializer extends Serializer {
  def toJson(x: Boolean): String = x.toString
}
{% endhighlight %}

Now our `Serializer` instance can serialize three types through a single logical `toJson` method.  Note that all we've done here is provide a new implementation of an exisitng named function.  We also had to use inheritance to do it, which muddies the water a bit, but I'll save editorial comments for another time.

If you wanted to do the same thing in Haskell you might use a typeclass.

{% highlight haskell %}
{-# LANGUAGE TypeSynonymInstances #-}

class Json a where
  toJson :: a -> String

instance Json String where
  toJson x  = "\"" ++ x ++ "\""

instance Json Int where
  toJson x  = show x

instance Json Bool where
  toJson True  = "true"
  toJson False = "false"

main :: IO ()
main = do putStrLn $ toJson "Hi there"
          putStrLn $ toJson 42
          putStrLn $ toJson false

{% endhighlight %}

This gives us a polymorphic function `toJson` that works with `String`, `Int`, and `Bool`.  Again we've just provided multiple implementations of the same named function and the correct implementation is chosen based on the argument type -- i.e. it's just overloading.

For fun, here it is with a Clojure protocol:

{% highlight clojure %}
(defprotocol Json
  (to-json [x] "Serialize x to a JSON string"))

(extend String
  Json
  {:to-json (fn [x] (str "\"" x "\""))})

(extend Integer
  Json
  {:to-json (fn [x] (str x))})

(extend Boolean
  Json
  {:to-json (fn [x] (str x))})

(println (to-json "Hi there"))
(println (to-json 42))
(println (to-json false))
{% endhighlight %}

The protocol is just a mechanism for overloading the function `to-json`.  You could also do this as a Clojure multimethod, but we'll leave that for now.

### Coercion

I didn't really want to cover coercion, but I feel obligated since I've talked about everything else.  With coercion, a function doesn't have to be written to work with different types, but instead, when applied the system automatically converts the arguments to the types required by the function.

This is considered ad-hoc polymorphism becuase each coercion has to be explicitly handled with custom code.

A silly Javascript example from [here][8]:

{% highlight javascript %}
7 + 7 + 7; // = 21  
7 + 7 + "7"; // = 147  
"7" + 7 + 7; // = 777  
{% endhighlight %}

The `+` function does numeric addition until it encounters a string argument, at which point it converts (coerces) the other argument to string and starts doing string concatenation.

More common examples from Java:

{% highlight java %}
(1 + 4) / 2   // 2 -- everything is an int
(1 + 4) / 2.0 // 2.5
double x = 5; // works, despite 5 reading as an int
{% endhighlight %}

In the second line the divisor is a double, so the dividend is coerced to a double before the division is done.  The assignment operation for variable `x` also works, despite the right side parsing as an integer.  Java coerces 5 to a double before assigning.

Scala is an interesting case here because it supports user-created [implicit conversions][9].  These could be used to coerce arguments before a function is applied.  I'll leave further investigation as an exercise for the reader.

## Polymorphic Functions vs. Types

This is particularly pedantic, even by the standards of this article!

Polymorphic function
: A function that accepts arguments that can have more than one type.

Polymorphic type
: A type with associated operations (i.e. methods for OO) that can be applied to values that have more than one type.

## Compile-time vs. runtime Polymorphism

## Summary

## Conclusion


[]: http://www.cis.upenn.edu/~bcpierce/tapl/ "Types an dProgramming Languages by Benjamin C. Pierce"
[]: http://www.itu.dk/courses/BPRD/E2009/fundamental-1967.pdf "Fundamental Concepts in Programming Languages, Strachey, 1967"
[1]: http://lucacardelli.name/Papers/OnUnderstanding.A4.pdf "On Understanding Types, Data Abstraction, and Polymorphism, Cardelli and Wegner, 1985"
[2]: http://en.wikipedia.org/wiki/Polymorphism_(computer_science) "Polymorphism at Wikipedia"
[3]: http://stackoverflow.com/a/367396 "What is the difference between Abstraction and Polymorphism"
[4]: http://stackoverflow.com/a/6885445 "Parametric polymorphism vs ad-hoc polymorphism"
[5]: http://stackoverflow.com/questions/5854581/polymorphism-in-c/5854862#5854862 "Polymorphism in C++"
[6]: http://www.haskell.org/tutorial/classes.html "Haskell tutorial on type classes"
[7]: http://en.wikipedia.org/wiki/Inclusion_(set_theory)
[8]: http://blog.jeremymartin.name/2008/03/understanding-loose-typing-in.html "Weak typing in Javascript"
[9]: http://www.codecommit.com/blog/ruby/implicit-conversions-more-powerful-than-dynamic-typing "Scala implicit conversion"
[10]: http://en.wikipedia.org/wiki/Subtype_polymorphism "Subtype Polymorphism"
[12]: http://www.tfeng.me/papers/milner78theory.pdf "A Theory of Type Polymorphism in Programming, Milner, 1978"

## Answers to Everything

Most of this article is based on this [fine paper on understanding polymorphism][1] (pdf).  By all means skip my article and just read section 1.3 of the paper!  Or stay tuned since I'll throw in some code examples and filter some of the details.  The entire first section of the paper is very accessible, but sections two and beyond are perhaps... not for everyone.

## Confusion With Other Things

I found that a lot of online discussion seemed to equate the following terms with polymorphism:

* dynamic dispatch
* interface and implementation inheritance
* data abstraction

These concepts _are_ related to polymorphism 

