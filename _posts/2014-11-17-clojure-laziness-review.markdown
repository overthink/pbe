---
layout: post
title: Clojure Laziness Review
description: A quick review of lazy sequences in Clojure
---

Quick notes on Clojure lazy sequences since I have confused myself on this a
few times over the years, and just did so again.

The working example will be a hack implementation of [`range`](https://clojuredocs.org/clojure.core/range).

{% highlight clojure %}

(defn range-so
  "Will blow stack."
  ([end] (range-so 0 end))
  ([start end]
  (when (< start end)
    (cons start (range-so (inc start) end)))))

;; (range-so 10000)
;; StackOverflowError   clojure.lang.Numbers$LongOps.inc (Numbers.java:529)

{% endhighlight %}

`range-so` is the naive recursive implementation that consumes stack and is not
useful for ranges with more than a few thousand entries.  It's also eager, so
`(take 2 (range-so 1000))` will calculate the full item range before returning
just the first two elements.

Clojure provides `recur` for non-stack-consuming "recursion", so that will solve
the `StackOverflowError`:

{% highlight clojure %}

(defn range-recur
  "Won't blow stack due to recur."
  ([end] (range-recur [] 0 end))
  ([start end] (range-recur [] start end))
  ([acc start end]
   (if (< start end)
     (recur (conj acc start) (inc start) end)
     acc)))

;; (range-recur 10000)
;; [0 1 .. 9998 9999]

{% endhighlight %}

`range-recur` is also eager, so again `(take 2 (range-recur 1000)` realizes the
whole range before returning only the first two items.

To make this lazy the obvious tool is the
[`lazy-seq`](https://clojuredocs.org/clojure.core/lazy-seq) macro.  Often you
can just wrap your seq-bearing expressions in `lazy-seq` and you're set.  One
caveat: don't blindly wrap code using `recur` with `lazy-seq` of you'll get
potentially surprising errors:

{% highlight clojure %}

(defn range-lazy-busted
  "Won't compile."
  ([end] (range-recur [] 0 end))
  ([start end] (range-recur [] start end))
  ([acc start end]
   (lazy-seq
     (if (< start end)
       (recur (conj acc start) (inc start) end)
       acc))))

;; IllegalArgumentException: Mismatched argument count to recur, expected: 0 args, got: 3

{% endhighlight %}

The reason for this is obvious when you realize that `lazy-seq` is just a macro
that wraps its contents in `(new clojure.lang.LazySeq (fn* [] ...)`.  The
surrounding `fn*` is a recur target expecteing zero args, hence the error
message.  Also, when you realize you're returning a `LazySeq` instance
immediately, it should be clear that you don't need `recur` since there is no
recursion.

Here's the fully lazy version:

{% highlight clojure %}

(defn range-lazy
  "Won't blow the stack due to lazy-seq."
  ([end] (range-lazy 0 end))
  ([start end]
   (lazy-seq
     (when (< start end)
       (cons start (range-lazy (inc start) end))))))

;; (range-lazy 10000)
;; (0 1 .. 9998 9999)

{% endhighlight %}

tl;dr - Recursive-looking self-calls ok when wrapped in `lazy-seq`.

