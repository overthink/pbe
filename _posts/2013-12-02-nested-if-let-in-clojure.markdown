---
layout: post
title: Nested if-let in Clojure
---

At work today some code turned up that looked like this:

{% highlight clojure %}
;; sum a, b, c, d if they're all defined, otherwise report which one is not
;; defined
(if-let [a x]
  (if-let [b y]
    (if-let [c z]
      (if-let [d w]
        (+ a b c d)
        :d-is-nil)
      :c-is-nil)
    :b-is-nil)
  :a-is-nil)
{% endhighlight %}

This is not easy to read, even in in the vastly simplifed form shown above.
The `else` expressions are far away form their corresponding `then`
expressions.

I couldn't find a good built-in or idiom for doing this in Clojure.  For reference, here's what I looked at:

* [Better way to nest if-let in clojure](http://stackoverflow.com/q/11673825/69689)
* [Why don't when-let and if-let support multiple bindings by default?](http://stackoverflow.com/q/11676120/69689)
* [when-let maybe?](http://inclojurewetrust.blogspot.ca/2010/12/when-let-maybe.html)

Since it's extremely repetitive looking code, it seemed like a macro might
be in order.  I came up with this:

{% highlight clojure %}
(defmacro nif-let
  "Nested if-let. If all bindings are non-nil, execute body in the context of
  those bindings.  If a binding is nil, evaluate its `else-expr` form and stop
  there.  `else-expr` is otherwise not evaluated.

  bindings* => binding-form else-expr"
  [bindings & body]
  (cond
    (= (count bindings) 0) `(do ~@body)
    (symbol? (bindings 0)) `(if-let ~(subvec bindings 0 2)
                              (nif-let ~(subvec bindings 3) ~@body)
                              ~(bindings 2))
    :else (throw (IllegalArgumentException. "symbols only in bindings"))))
{% endhighlight %}

I'm not an experienced macro writer.  Most of this is adapted from
[`with-open`](https://github.com/clojure/clojure/blob/c6756a8bab137128c8119add29a25b0a88509900/src/clj/clojure/core.clj#L3442)
which felt vaguely similar in that it generates a deeply nested bunch of code.
It also will only work for simple binding forms, but it was good enough for the
situation at hand.  The nested code above becomes:

{% highlight clojure %}
(nif-let [a x :a-is-nil
          b y :b-is-nil
          c z :c-is-nil
          d w :d-is-nil]
         (+ a b c d))
{% endhighlight %}

This reads a lot better, but having to ensure the result is not some arbitrary set of
keywords to determine that it actually worked is non-ideal.  Slightly better
might be to wrap the result in something more descriptive.

{% highlight clojure %}
(let [fail (fn [x] {:status :failed :value x})
      ok (fn [x] {:status :ok :value x})]
  (nif-let [a x (fail :a-is-nil)
            b y (fail :b-is-nil)
            c z (fail :c-is-nil)
            d w (fail :d-is-nil)]
           (ok (+ a b c d))))
{% endhighlight %}

This will yield values like `{:status :failed :value :a-is-nil}` or `{:status
:ok :value 42}`, which is slightly less bad to inspect.  You could easily
include the `fail` and `ok` calls in the macro itself, but it seems more
reusable not to.

This all has a monadic smell to it.  I'm sure there's an existing monad in
[`algo.monads`](https://github.com/clojure/algo.monads) that could do it
succinctly, but I'd want to spend some time understanding that code before
pulling it in.

I find it odd that there's not a common idiom in the "standard libarary" for
this.  Maybe I just haven't been searching for the right thing.

**Update 2013-12-12**: [solution to this same problem in Scala]({% post_url 2013-12-12-combining-option-and-either-in-scala %}).

