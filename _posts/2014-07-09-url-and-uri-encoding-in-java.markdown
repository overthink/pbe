---
layout: post

title: URL Encoding, Percent-Encoding and Query Strings in Java
description: Refresher on the gory details of correct URL encoding in Java.
published: true
---

I've been lost in the details of URL encoding a number of times.  Each time I
figure it out, move on, then promptly forget everything about it.  This article
is nowhere near a complete reference, it exists mainly to jog my memory.

For the rest of the article I'm going to say __URI__ instead of URL, but I'm
really thinking about URLs -- http and https schemes in particular.

The majority of this info comes from
[RFC3986](http://tools.ietf.org/html/rfc3986) and the Wikipedia page on
[percent-encoding](http://en.wikipedia.org/wiki/Percent-encoding).

## High level

* There's nothing in the JDK to make correct URI encoding/decoding simple
* There are various half-measures in the JDK that will get you occasionally-correct
  code (`URI`, `URLEncoder`, `URLDecoder`) that might pass a smoke test
* This stuff only _looks_ easy, there are a million edge cases
* __Consider a library__
  * [urlbuilder](https://github.com/mikaelhg/urlbuilder) (small and to the point; doesn't support matrix params?)
  * [jersey's URIBuilder](https://jersey.java.net/apidocs/2.0/jersey/javax/ws/rs/core/UriBuilder.html) (huge dependency)
  * [apache httpclient's URIBuilder](http://hc.apache.org/httpcomponents-client-ga/httpclient/apidocs/org/apache/http/client/utils/URIBuilder.html) (huge dependency)
* If rolling your own
  * Each component of a URI has different rules for which characters need
    percent-encoding
  * Encode each component of a URI separately, applying component-specific
    percent-encoding rules
  * For decoding, [regex parse](http://tools.ietf.org/html/rfc3986#appendix-B)
    incoming URI string into components, then decode each separately using
    component-specific rules
* __percent-encoding is different than `application/x-www-form-urlencoded`__
  * They differ in handling of spaces and newlines
* The query string is typically encoded using
  `application/x-www-form-urlencoded`, but doesn't have to be according to the
  URI spec.

## URI Components

A URI is made up of five components.  So called "reserved characters" are used
to separate the components from one another. From RFC3986:

     foo://example.com:8042/over/there?name=ferret#nose
     \_/   \______________/\_________/ \_________/ \__/
      |           |            |            |        |
    scheme     authority       path        query   fragment

The five components are scheme, authority, path, query, and fragment.  Some
components can be broken into sub-components (my term).  For example the path
component `over/there` has two sub-components: `over`, and `there`, separated
by the reserved charcter `/` (at least for the http(s) scheme).  Authority has
host and port sub-components, and uses the reserved character `:` to separate
them.  Query has no sub-components.  It's just everything from the first `?` to
the first following `#` or the end of the URI.  Typically the data in query is
HTML form encoded.

Why care about components and sub-components?  __Each sub-component needs to be
percent-encoded using a component-specific set of reserved characters, then
assembled with the correct URI syntax__.  (e.g.
[path](http://tools.ietf.org/html/rfc3986#section-3.3) vs.
[query](http://tools.ietf.org/html/rfc3986#section-3.4)) In the standard JDK
there are no foolproof shortcuts for doing this.  You have to break your URL
into it's smallest components, percent-encode each part, then assemble with the
appropriate syntax.

From ad-hoc testing, it seems that implmentations of URI and form data parsing
are very lenient and even if you percent-encode something that doesn't require
it, you'll probably get the result you want.

## URLEncoder and URLDecoder

These are horribly named classes.  You might expect them to be useful for
encoding a URL.  They're not.  They are good for __HTML form encoding__,
though.  These classes encode and decode the
`application/x-www-form-urlencoded` MIME type, which is similar to what you
need for general purpose URI encoding, but not quite.  These classes don't know
anything about which component of a URI you're working on, and they also always
encode spaces to `+` (instead of `%20`), which is only a valid encoding for
HTML form data (i.e. in the query string only).  You can use these classes as a
blunt instrument to percent-encode strings, but you'll need component-specific
fixups after encoding depending on what part of a URI you're working on.  Not
an attractive option.

## URI Class

The URI class gets some parts almost right.  Its
[source](http://grepcode.com/file/repository.grepcode.com/java/root/jdk/openjdk/6-b14/java/net/URI.java#URI)
is an interesting read, though.  It's where I first learned about different
components having different reserved characters.  If you use the many-argument
constructors for `URI`, the correct encoding rules are applied sometimes,
depending on your data.

{% highlight clojure %}
(java.net.URI. "http" "user:password" "foo.com" 8080 "/foo/bar/a+b/c d/baz" "a=Mark's stuff&c=yo" "frag")
;; http://user:password@foo.com:8080/foo/bar/a+b/c%20d/baz?a=Mark's%20stuff&c=yo#frag
{% endhighlight %}

* path looks good in this example
  * it did the right thing preserving the `+` and using `%20` for spaces
  * it understood that `/` is a path separator
  * you're screwed if your path element contains a "/" that needs encoding though
* query is wrong, but likely works in practice
  * it's percent-encoded, not form encoded (note the apostrophe is not
    percent-encoded, this is fine by URI spec, but not by form encoding spec)
  * it didn't encode `=` and `&` which is fine unless one of your values is
    `a & b == c`, in which case you're screwed again

Using `URI` to do general purpose URL encoding is not a viable option.  It does
look like it's a usable URI parser though, using the single-string constructor
and the `raw` getters.

## Conclusion

* Use a library
* Write your own URI encoder/decoder helpers, but recognize it's less
  straight-forward than you probably want it to be
* Hope for a working URIBuilder in the JDK one day
* A standard set of tests for URI parsing would be nice, too

## References

* [Detailed discussion of URL encoding with examples](http://blog.lunatech.com/2009/02/03/what-every-web-developer-must-know-about-url-encoding) (Excellent read)
* [RFC3986 - URI: Generic Syntax](http://tools.ietf.org/html/rfc3986)
* [percent-encoding](http://en.wikipedia.org/wiki/Percent-encoding) at Wikipedia
* [application/x-www-form-urlencoded "spec"](http://www.w3.org/TR/html401/interact/forms.html#h-17.13.4.1) (as best I can find...)
* [Tests for URI parsing](http://www.w3.org/wiki/UriTesting)
* [Python's urlparse](https://docs.python.org/2/library/urlparse.html)

