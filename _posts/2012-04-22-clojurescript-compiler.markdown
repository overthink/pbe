---
layout: post
title: A Look at the ClojureScript Project
published: false
---

## Intro

This article begins a detailed look at the [ClojureScript](https://github.com/clojure/clojurescript) project.  The overall goals are:

1. Understand how the compiler works
1. Learn more about the design and structure of a non-trival Clojure project

For this first post I'll:

1. Clarify terminology
1. Look at how the source tree is organized
1. Determine what is actually happening when you use a ClojureScript repl (`script/repljs`) or invoke the compiler (`bin/cljsc`)

## Definitions

Clojure, Closure, ClojureScript.  It is easy to get lost in terminology when talking about ClojureScript.  I'll use the following definitions in this article:

Clojure Language
: [There is no Clojure language spec](http://stackoverflow.com/questions/3902813/is-there-a-language-spec-for-clojure), but if there was, that is what I'd refer to here!  Roughly: the [reader forms and macros](http://clojure.org/reader), [special forms](http://clojure.org/special_forms) and [core API](http://clojure.github.com/clojure/clojure.core-api.html) of the original Clojure (JVM).

Clojure
: A compiler (written in Java) for the Clojure Language producing JVM bytecode.  This is what people are talking about 99.9% of the time when they say *[Clojure](http://clojure.org)*.

ClojureScript
: A compiler (written in Clojure) for the Clojure Lanugage producing JavaScript.

Google Closure
: [Google Closure](https://developers.google.com/closure/).  An optimizing JavaScript compiler and collection of other tools aiming to make JavaScript development less bad.  ClojureScript relies on Closure's JavaScript compiler to generate small, high-performance JavaScript code.

## Project Layout

ClojureScript is implemented in both Clojure (`src/clj`) and ClojureScript (`src/cljs`).

- `src/clj/cljs/` - Clojure (JVM) code
  - `core.clj`
    - defines macros that redefine a huge amount of Clojure's core functions
  - `compiler.clj`
    - requires: core.clj
    - analyze functions seem important
  - `closure.clj`
    - requires: compiler.clj
    - uses ClojureScript compiler (compiler.clj) and Google Closure to build the final js source
    - defines IJavaScript and Compilable protocols
      - an IJavaScript contains actual js source, requires/provides info (on namespaces)
    - 'build = compile, add-dependencies, optimize, output'
      - compile - return IJavaScripts
      - add-dependencies - get deps as IJavaScripts for all given IJavaScripts, in dep order
      - optimize - give code to Google Closure to do the hard work
    - talks about "classpath" dependencies in here, not sure how they matter
  - `repl.clj`
    - requires: compiler.clj, closure.clj
  - `repl/` - Not sure yet
- `src/cljs/` - ClojureScript code
- `bin/` - Helper scripts
  - `cljsc` - Wrapper `sh` script to invoke `cljsc.clj`
  - `cljsc.clj` - Tiny Clojure (JVM) app that invokes the Google Closure wrapper (`cljs.closure/build`)

## ClojureScript REPL

## ClojureScript Compiler

## References / Reading List

These are listed in the order I think you should read them.

1. [ClojureScript Rationale](https://github.com/clojure/clojurescript/wiki/Rationale) - Rich Hickey explains why ClojureScript came to be.
1. [The REPL and Evaluation Environments](https://github.com/clojure/clojurescript/wiki/The-REPL-and-Evaluation-Environments) - There can be different Es in the ClojureScript REPL
1. [The Himera Model](http://blog.fogus.me/2012/03/27/compiling-clojure-to-javascript-pt-3-the-himera-model/) - A look at the modules that make up the ClojureScript compiler (and a cool browser REPL)
1. [The ClojureScript Compilation Pipeline](http://blog.fogus.me/2012/04/25/the-clojurescript-compilation-pipeline/) - More on the modules of the ClojureScript compiler: "between each compilation phase the interface consists solely of Clojure data"
1. [ClojureScript source](https://github.com/clojure/clojurescript/tree/bf2c682398101bdc93d2f97cdad410cb214987f6) - The particular version of the code I looked at while writing this article.

