---
layout: post
title: Print and read in Clojure
description: Serializing forms safely with print and read in Clojure.
---

I recently had a situation where I needed a very light-weight cache for some
data that took forever to come back from a slow external API.  My requirements:

* cached data must outlive current process, i.e. be written to disk
* cache must throw at write time if the thing being written won't be readable
  when we need it
* it's ok to let `read` evaluate code (i.e. trust values from the disk cache)
* no databases, bloated dependencies, just quick and dirty

Given that I was using Clojure, I figured this would be extremely easy given
the Lisp dream of pervasive print/read to/from textual forms.  The high-level
implementation is use
[`pr`](http://clojure.github.io/clojure/clojure.core-api.html#clojure.core/pr)
to print values to a text file, and
[`read`](http://clojure.github.io/clojure/clojure.core-api.html#clojure.core/read)
to read values back into memory.  I encountered some rough edges in Clojure's
print/read in doing this, however.

## Printing

The first issue I hit is that `pr` and its related helpers like `pr-str` will
happily print anything, even if it has no hope of being read back in. e.g. A
`org.joda.time.DateTime` object has no readable text form by default:

{% highlight clojure %}
(read-string (pr-str (org.joda.time.DateTime.)))
;; RuntimeException Unreadable form
;;         clojure.lang.Util.runtimeException (Util.java:219)
;;         clojure.lang.LispReader$UnreadableReader.invoke (LispReader.java:1116)
;;         clojure.lang.LispReader$DispatchReader.invoke (LispReader.java:626)
;;         clojure.lang.LispReader.read (LispReader.java:185)
;;         clojure.lang.RT.readString (RT.java:1738)
;;         clojure.core/read-string (core.clj:3427)
{% endhighlight %}

There's nothing special about `DateTime`, it's just my example.  No Java class
(except a handful with special handling in Clojure) will have readable printed
forms out of the box.  That's fine, but I do want to **know** when something
I'm printing will not be readable.

There is a solution to this though: the dynamic variable `*print-dup*`.  I
assume `dup` is short for dupliacte?  First, background: The print-related core
functions boil down to calls to
[`pr-on`](https://github.com/clojure/clojure/blob/201a0dd9701e1a0ee3998431241388eb4a854ebf/src/clj/clojure/core.clj#L3386-L3393)
which calls the multimethod `print-method` in the usual case, or another
multimethod `print-dup` if `*print-dup*` is true.  Here's the code:

{% highlight clojure %}
(defn pr-on
  {:private true
   :static true}
  [x w]
  (if *print-dup*
    (print-dup x w)
    (print-method x w))
  nil)
{% endhighlight %}

`print-method` has an [implementation for
`java.lang.Object`](https://github.com/clojure/clojure/blob/201a0dd9701e1a0ee3998431241388eb4a854ebf/src/clj/clojure/core_print.clj#L101-L102),
which is why it is able to print anything.  `print-dup`, on the other hand has fewer
stock methods, and will throw when you try to print something it doesn't
explicitly know about.  This is what I wanted: if it won't be readable, blow
up.  (Note my reasonable? assumption here that if a type has a `print-dup` implementation
that it will be readable.)

{% highlight clojure %}
(binding [*print-dup* true] (pr-str (org.joda.time.DateTime.)))
;; IllegalArgumentException No method in multimethod 'print-dup' for dispatch value:
;; class org.joda.time.DateTime  clojure.lang.MultiFn.getFn (MultiFn.java:160)
{% endhighlight %}

This is great, except that to be useful -- at least for me -- certain non-Clojure
objects, like joda's `DateTime` need to work with the cache.  Since `print-dup`
is a multimethod, we can easily extend it for our uses.

{% highlight clojure %}
;; Make it possible to print joda DateTime instances; required for caching.
(defmethod print-dup org.joda.time.DateTime
  [dt out]
  (.write out (str "#=" `(DateTime. ~(.getMillis dt)))))
{% endhighlight %}

Now the following works:

{% highlight clojure %}
(binding [*print-dup* true] (pr-str (org.joda.time.DateTime.)))
"#=(org.joda.time.DateTime. 1395949236863)"
{% endhighlight %}

## \*read-eval\*

Notice that I use `#=` in the output here.  This is an undocumented, but well
used, reader macro that causes the following form to be evaluated after it is
read (i.e. at read time), but only if `*read-eval*` is true (which it is by
default).  This feels a bit scary on the surface -- an attacker could plant
`#=(fire-missiles :now)` in the cached data -- but I'm accepting it.  In my
case, a user that can modify the cached data already has the same access level
as the program does.  See references below for some more on `*read-eval*` safety.

In general, you need to ensure `*read-eval*` to read things printed by
`print-dup`. Search for `defmethod print-dup`
[here](https://github.com/clojure/clojure/blob/201a0dd9701e1a0ee3998431241388eb4a854ebf/src/clj/clojure/core_print.clj)
to see how often it is used in the stock methods for `print-dup`.

Things appear to be printing.  Now for the read side.

## Reading

Most of the complexity I encountered was on the print side, so reading is
pretty easy: just make sure `*read-eval*` is set:

{% highlight clojure %}
(binding [*read-eval* true] (read-string (pr-str org.joda.time.DateTime.)))
;; #<DateTime 2014-03-27T15:40:36.863-04:00>
{% endhighlight %}

## y u do dis struct-map?

I wrote all this up, using `pr` and `read` to read/write files on disk, and
started using it.  Right away things blew up: `IllegalArgumentException No
matching method found: create`.  WTF?  Looking at the cached data I saw tons of
this sort of thing:

{% highlight clojure %}
#=(clojure.lang.PersistentStructMap/create {:a 1, :b 2})
{% endhighlight %}

[`struct-map`s](http://clojure.org/data_structures#Data%20Structures-StructMaps).
Old, and largely replaced by records.  Who the hell uses struct maps?  I
don't...?  er, But I do use
[`resultset-seq`](http://clojure.github.io/clojure/clojure.core-api.html#clojure.core/resultset-seq)
extensively, and it returns struct maps.  These things get through `print-dup`
because `PersistentStructMap` implements `IPersistentMap`, which is handled by
`print-dup`
[here](https://github.com/clojure/clojure/blob/201a0dd9701e1a0ee3998431241388eb4a854ebf/src/clj/clojure/core_print.clj#L216).
The fact that they can't be a read is an [ancient
bug](http://dev.clojure.org/jira/browse/CLJ-176) that doesn't look likely to be
fixed.  Ugh.

I found an easy work-around, and while it's not really in the
spirit of `print-dup`, it works for me: convert struct maps to regular maps
before printing:

{% highlight clojure %}
;; Convert struct maps into something readable before printing
;; http://dev.clojure.org/jira/browse/CLJ-176
(defmethod print-dup clojure.lang.PersistentStructMap
  [m out]
  (print-dup (into {} m) out))
{% endhighlight %}

I don't know if there are more instances of this type of thing (printable with
print-dup, but not readable); I haven't found any yet.  This cache doesn't see
a great variety of values, however.

## tl;dr

How to print things and be confident you can read them later:

* bind `*print-dup*` to true at print time
* add `print-dup` methods as necessary for your custom and 3rd party types
* always add a custom `print-dup` method for `PersistentStructMap`
* set `*read-eval*` to true at read time

References

* [Don't use XML/JSON for Clojure-only persistence/messaging](http://amalloy.hubpages.com/hub/Dont-use-XML-JSON-for-Clojure-only-persistence-messaging)
* [struct-maps aren't readable](http://dev.clojure.org/jira/browse/CLJ-176)
* [`*read-eval*` safety discussion, decisions](https://groups.google.com/forum/#!topic/clojure/qUk-bM0JSGc)

