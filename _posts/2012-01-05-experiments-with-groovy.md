---
layout: post
title: experiments with groovy
date: 2012-01-05 17:30
comments: true
tags: groovy code
published: true
---

In my constant task to hone and resharpen my tools, I've started some experiments
with **groovy**.

<!-- more -->

Groovy is a scripting language which runs on top of the *JVM* like *Jython* or *jRuby*.  But
unlike other scripting languages, **groovy** classes are full-blown Java classes and can be used
within plain Java.  Also, any Java package can be used in groovy without writing bridges and
stuff.

Also it seems that **groovy** fits my python and coffee-script infested brain better than the more
modern and hip cousins **clojure** and **scala**.

This looks pretty readable to me!

classes, closures, iterators -- oh my
-------------------------------------

**groovy** is fully OO -- grok this:

```groovy
def map = [:]

map.some_key = "value"
map.another_key = "foo"
map["yet another key"] = "bar"

map.each { item ->
	println "$item.key => $item.value"
}

// implicit maps
def errors = [ EINVAL: -1, ENOSPC: -3, EPROTO: -42] // just some example
println errors.EINVAL  // -1
```

The block enclosed in curly braces is a *closure*.  They're objects, too:

```groovy

// a closure
def closure = { a, b -> a + b }

// let's call it
println closure.call(1, 2)  // 3

// same, less verbose
println closure(1, 2)  // 3

// we can also accept a closure as a parameter
def when(condition, closure) {
	if (condition)
		closure()
}

when (4<5) {
	println "yay!"
}

def local = 2
def test = { x -> local + x }

println "test(2) => ${test(2)}" // test(2) => 4
local = 3
println "test(2) => ${test(2)}" // test(2) => 5

def test2 = { x ->
	def local2 = x
	return { k -> k + local2 }
}

def test_2 = test2(2)
def test_3 = test2(3)

println "test_2(2) => ${test_2(2)}" // test_2(2) => 4
println "test_3(2) => ${test_3(2)}" // test_3(2) => 5
```

Another nice thing is how classes are defined and how **groovy** creates automatic
constructors:

```groovy
// a simple class.   Note that we do not define a constructor.
class Person {
	public String name
	public Integer age
}

def anonymous = new Person()
def stefan    = new Person( name: "Stefan", age: 38)
def fritz     = new Person( name: "Fritz")

println "${stefan.name} is ${stefan.age} years old." // stefan is 38 years old.
println "${fritz.name} is ${fritz.age} years old." // fritz is null years old.

stefan.age = 35 // no setters and getters needed!
println "${stefan.name} is ${stefan.age} years old." // stefan is 35 years old.

// coerce a map to a class.  Will call named-arg automatic ctor
def maja = [name:"Maja", age:8] as Person
println "${maja.name} is ${maja.age} years old." // maja is 8 years old.

```

Notice how we did **not** need to specify getter and setter methods.  Also notice
how **groovy** uses named arguments!

What's next?
------------

Well.  Next up, I'll try to get some `PDMlink` stuff working with **groovy**.

Links
-----

You can find **groovy** documentation at [codehaus] [1].  There's also a zone
over at [dzone] [2].

There's also a API documentation of the [groovy JDK] [3].

  [1]: http://groovy.codehaus.org/      "Groovy at codehaus"
  [2]: http://groovy.dzone.com/  	"Groovy Zone"
  [3]: http://groovy.codehaus.org/api/overview-summary.html  	"Groovy JDK"


