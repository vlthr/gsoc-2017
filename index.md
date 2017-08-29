---
date:   2017-08-29 11:49:45 +0200
title: "A Summer of Scala"
author: Valthor Halldorsson
categories: gsoc, scala, dotty
---
# A Summer of Scala
Over the past few months I’ve had the opportunity to work with the Scala organization as part of the Google Summer of Code program. It all started when Felix Mulder held a guest lecture at my university's main compiler course, showing off all the cool things the Scala team were doing with Dotty. I was immediately interested, but as so often happens with interesting things it ended up being pushed to the side with all the other interesting things that you never quite have time to learn more about. It wasn't until about a year later when I heard that the Scala organization was looking for applicants to their GSoC program that I finally got in touch with Felix and asked how I could contribute. After picking an open issue on Dotty's tracker and submitting a couple of pull requests, we discussed a bunch of ideas for possible GSoC projects. I picked one, put together a proposal, submitted it, and it got accepted!

When summer finally came, some of the groundwork required for my proposal to be implemented was still unfinished, and rather than taking the risk of needing to rewrite the project by the end of the summer, we decided to tackle a different problem instead: Writing a template rendering engine using Dotty. The aim of the project was twofold. First, to test the compiler on a medium sized project by finding and reproducing error cases. Second, to use the templating engine to provide stricter compile-time safety for DottyDoc, Dotty’s built in documentation generator.

## Why Dotty?

Dotty is the experimental new compiler the Scala team is working on to one day replace scalac as the main Scala compiler. The reasons for doing a complete rewrite are varied, but in short, Dotty's type system is built on the newly developed [DOT calculus](http://www.scala-lang.org/blog/2016/02/03/essence-of-scala.html), giving it firmer theoretical underpinnings. The compiler now also splits its work into more distinct mini-phases which can be parallellized to a greater extent than before, allowing for faster compile times. These may not be the first thing an end user thinks about when using the compiler but they give rise to a host of implications for reliability, performance and ease of development.

Apart from these big picture issues, Dotty also brings a range of new features that are more visible to the end user. The complete list can be seen [on the Dotty homepage](http://dotty.epfl.ch/#so-features) but here are some of my favorites:
- Trait parameters: Traits can now take parameters using the same syntax as for classes. While it has always been possible to work around this limitation in Scala2 by either converting the trait to a class (and losing multiple inheritance) or declaring the parameter as an abstract member on the trait, the reduction in boilerplate and increase in consistency is a welcome change. 
- Union types: Dotty allows you to express types of the form `A | B`, meaning `A or B`. When I first read about them, I couldn't really see myself using them. What surprised me was that given the option, union types a very simple and easy to use tool, and I found myself reaching for them on multiple occasions. For instance:
```scala
def write(content: String, destination: String | File | Writer): Unit = {
  val writer = destination match {
    case f: File => new PrintWriter(new BufferedWriter(new FileWriter(f)))
    case f: String => new PrintWriter(new BufferedWriter(new FileWriter(f)))
    case w: Writer => w
  }
  writer.write(content)
}
```
Moreover, because `A | B` is a subtype of any other union that contains both `A` and `B`, you can pass the union to other methods as-is:
```scala
def makeWriter(f: String | File | Path) = new PrintWriter(new BufferedWriter(f match {
  case f: File => new FileWriter(f)
  case f: String => new FileWriter(f)
  case f: Path => new FileWriter(f.toFile)
}))

def write(content: String, destination: String | File | Writer): Unit = {
  val writer = destination match {
    case f: (String | File) => makeWriter(f)
    case w: Writer => w
  }
  writer.write(content)
}
```

While union types often partially overlap with the use cases for method overloading, complicated class hierachies, or abstractions like Either they are very easy to use and require minimal overhead.
- Automatic tupling: To map a function over a list of tuples in Scala2, you have to pattern match on the tuple. Dotty allows for automatic unpacking of tuples when used as function arguments.
```scala
val tuples: List[(String, String)] = ???

// Scala2
tuples.map {
  case (a, b) => a+b
}
// Dotty
tuples.map((a, b) => a+b)
```
While the difference is small, I find the Dotty style easier to parse and more consistent with my expectations.

- Improved type inferencing: This one is hard to pin down as it's not a single identifiable feature like the rest. On the whole, Dotty's type system is much better at inferring the correct types for more complex expressions. This became especially noticable when I started porting my Dotty code to Scala2 and had to add type annotations in places where I had just gotten used to the compiler figuring things out on its own. For example, the following compiles with Dotty, but requires either the `val` or the empty map constructor to be annotated in Scala2.
```scala
def printMap(m: Map[String, String]) = println(m)

val map = if (true) Map("hello" -> "world")
          else Map()

printMap(map)
```

## Show and tell
The end result is [Levee](https://github.com/vlthr/levee), a standalone templating engine for the Liquid templating language. The library is now available on Sonatype. In addition to providing strict rendering of Liquid templates, Levee allows users to define type safe filters without requiring the user to handle type checking.

```scala
// For the liquid filter {{ "World" | prepend: "Hello " }}
// Filter takes three type arguments for the Input, Args, and Optional Args respectively
// HNil is the type equivalent of a Nil, indicating the end of a list of types
val prepend = Filter[StringValue, StringValue :: HNil, HNil]("prepend") {
  (ctx, filter, input, args, optArgs) =>
    val prefix = args.head.get
    success(StringValue(prefix + input.get))
}
```

It was this custom filter feature that ended up taking most of my time, since for a long time every solution I could come up with required the user to either manually handle the type checking, or do some sort of unsafe casting of the inputs to the filter. After weeks of trying I was introduced to Shapeless, a library that provides a set of tools for doing generic programming in Scala. It took a few days to wrap my head around how the library works, but once it clicked I was finally able to solve the problem in a way I was happy with.

There is still some work to be done to make Levee feature complete, and I wouldn’t trust the code to run a space station, but with that said I'm pretty happy about how it turned out.

## Lessons learned
Looking back, there are a few things that I wish I had given more thought to early on in the project.
- Use a parser generator: Early on, I spent a fair amount of time writing a custom parser. It started out very simple, but complexity started piling on quickly as soon as I wanted features like detailed error reporting or error recovery. While it took a few days to learn how to use it, when I finally moved to using a parser generator (specifically ANTLR), things became a lot easier. The time I spent learning to use ANTLR has also paid off in the sense that I can very quickly whip up a parser for a new project, whereas my custom parser was very application specific. One caveat is that most parser generators rely on the lexer being able to produce a token stream independent of the parser, which may not be possible for all languages.
- Think about the way you want to handle errors: One of the core requirements for Levee was that if there are multiple errors in a template, the library should collect and report all of them together in a sane way. This makes error handling a lot more difficult than it otherwise might be, since you can't just throw an exception at the first sign of trouble. Early on I used `Either[List[Error], T]` to represent my results, but this resulted in a lot of boilerplate when there were multiple independent points of failure within a function. What I didn't realise at first was that there are much more powerful error handling abstractions available, such as [`Validated`](https://typelevel.org/cats/datatypes/validated.html) from cats, or scalaz's [`Validation`](http://eed3si9n.com/learning-scalaz/Validation.html). After switching over, virtually all of my error handling logic was replaced by easy to follow operations with the validation type.
- Don't be afraid of libraries like Shapeless. While at first glance code using Shapeless can seem like a unintelligible mess of arcane type incantations, the principles required to understand it are relatively few and easy to learn. If I had known how little time it would take for me to reach that understanding, I would have done it years ago. The biggest downside is that I now have to live with the knowledge that some of the most repetitive code I've written in my life could have been reduced to only a few lines.

Overall, my experience with Dotty was that for an experimental compiler it is surprisingly stable. Over the course of the summer I encountered a total of two compiler crashes and one OOM error, but out of these, two were related to some very unusual and experimental implicit resolution code, and only one of the crashes was caused by sound Scala code. While it may be too early to move your production codebase to Dotty, it may not be long until the expanded set of features makes the jump worthwhile.
