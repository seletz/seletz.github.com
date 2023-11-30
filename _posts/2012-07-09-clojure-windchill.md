---
layout: post
title: "Clojure for Windchill Development, Part 1"
date: 2012-07-09 10:06
comments: true
categories: clojure windchill
---

I always wanted to know more about functional programming and lisp -- so I
thought why not use [Clojure] [1].  It runs on the *JVM* and I can try out some
real-world problems with it.

Normally I use [Jython] [2] -- a implementation of the [Python] [3] programming
language which runs on the *JVM* -- to develop for Windchill.

[1]:  http://clojure.org
[2]:  http://jython.org
[3]:  http://python.org

This is the first part of a series of post's I'm going to make to log my progress.

In this part I'll show how to *authenticate* and *search* in Windchill using Clojure.

<!-- more -->

Installing
----------

I use a mac, so installing Clojure is quite easy for me -- YMMV::

```bash

    $ brew install clojure

```

Running the Clojure REPL
------------------------

That's easy, too:


```bash
    $ clj
    Clojure 1.4.0
    user=> (+ 1 2)
    3
```

Installing Clojure in Windchill
-------------------------------

This is quite easy, too.  Just drop the Clojure *jar* file to
`$WT\_HOME/codebase/WEB-INF/lib`.  Then, when starting the *REPL*, be sure to
have the `CLASSPATH` set up correctly.

First Steps using Windchill
---------------------------

The first Task I wanted to solve is:

- authenticate against Windchill using the `RemoteMethodServer` 
- search for a named `EPMDocument` using a custom-built `QuerySpec`

This is what I came up with:

```clojure
(import '(wt.method RemoteMethodServer)
        '(wt.epm EPMDocument)
        '(wt.fc PersistenceHelper)
        '(wt.query QuerySpec SearchCondition))

(defn authenticate [u p]
  (doto (. RemoteMethodServer getDefault)
    (.setUserName u)
    (.setPassword p)))

(defn sc-name [name]
  (SearchCondition. EPMDocument (EPMDocument/NAME) "LIKE" name false))

(defn make-query-spec [class]
  (QuerySpec. class))

(defn build-query-spec [name]
  (let [qs (make-query-spec EPMDocument)]
    (.appendWhere qs (sc-name name))
    qs))

(defn search [name]
  (enumeration-seq
    (.find PersistenceHelper/manager
      (build-query-spec name))))

;; Authenticate first
(authenticate "orgadmin" "orgadmin")

;; Do a search for `k05*.asm`
(def result
  (search "k05%.asm"))

;; and print out all the resulting objects.
(println result)
```

The code is also on [github] [4].

[4]:  https://github.com/seletz/clojure-windchill

Things To Note
--------------

  - The first few lines (1..4) import Classes from the named packages to the
    current namespace.

  - The `(doto instance expr)` special form saves quite some typing. (line 7)

  - It took a while until I figured out how to convert the enumeration from the
    `QueryResult` to a Clojure sequence.  It's the `(enumeration-seq expr)`
    special form.  This is quite cool, as this probably means that the results
    from the search would be processed lazily, too. (line 23)

  - I had quite some problems getting the `(let [...] expr)` special form right.
    For some reason I had one pair of parens too much and it kept evaluating a
    `nil` result from Java which caused a dreaded `Null Pointer Exception`.

  - I started to put the `(authenticate ...)` and the imports in a `user.clj`, such that the
    *REPL* has these aitomatically loaded.

Things to explore:

  - Use a custom name space.  Currently, all is in the `user` name space.

  - Use [leiningen] [5] to manage dependencies.

  - Better [VIM integration] [6]

[5]:  https://github.com/technomancy/leiningen/
[6]:  http://posterous.adambard.com/quickstart-clojure-on-vim-lein-slimv-windows
