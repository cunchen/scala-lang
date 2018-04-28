---
layout: blog-detail
post-type: blog
by: Martin Odersky and Nicolas Stucki
title: "Scala in a (Tasty) Nutshell"
---

One of the biggest open questions for migrating to Scala 3 is what to
do about macros. In this blog post we give our current thoughts. The
gist is that we are trying to achieve full alignment between macros
and Tasty.

## What is Tasty?

Tasty is the high-level interchange format for Scala 3. It is based on
<i>t</i>yped <i>a</i>bstract <i>s</i>yntax <i>t</i>rees. These trees
contain in a sense all the information present in a Scala
program. They represent the syntactic structure of programs and also
contain the complete information about types and positions. The Tasty
"snapshot" of a code file is taken after type checking (so that all
types are present and all implicits are elaborated) but before any
transformations (so that no information is lost or changed). The file
representation of these trees is heavily optimized for compactness,
which means that we can generate full Tasty trees on every compiler
run and rely on nothing else for supporting separate compilation.

The information present in Tasty trees can be used for many purposes.

 - The compiler uses it to support separate compilation.
 - Our LSP-based language server uses it to support hyperlinking, command completion, documentation,
   and also for global operations such as find-references and renaming.
 - A build tool can use it to cross-build on different platforms and migrate code from one binary
   version to another.
 - Optimizers and analyzers can use it for deep code analysis and advanced code generation

Among these use cases, the first two work today. The other two are
very interesting possibilities to pursue in the future.

OK, but what is Tasty _exactly_? An up-to-date version of the Tasty
file format is described in file
[TastyFormat.scala](https://github.com/lampepfl/dotty/blob/master/compiler/src/dotty/tools/dotc/core/tasty/TastyFormat.scala)
of the `dotc` compiler for Scala 3.

## What Does Tasty Have to Do with Macros?

It turns out that Tasty also makes an excellent foundation for a new
generation of reflection-based macros, with the potential to solving
many of the problems in the current version.

The first problem with the current `scala.reflect` macros is that they
are completely dependent on the current Scala compiler (internally
named `nsc`). In fact, `scala.reflect` macros are nothing but a thin
veneer on top of `nsc` internals. This makes them very powerful but
also fragile and hard to use. Because of this, they have had
"experimental" status for their whole lifetime. Since Scala 3 uses a
different compiler (`dotc`), the old reflect-based macro system cannot
be ported to it, so we need something different, and hopefully better.

Another criticism of `scala.reflect` macros is that they
lack _foundations_. Scala 3 has already a meta
programming facility, with particularly well explored foundations. [Principled Meta
Programming](http://dotty.epfl.ch/docs/reference/principled-meta-programming.html)
is a way to support _staging_ (in the sense of runtime code-generation)
by adding just two operators to the
language: Quote (`'`) to represent code expressions, and splice (`~`)
to insert one piece of code in another. The inspiration for our
approach [comes from temporal
logic](https://ieeexplore.ieee.org/abstract/document/561317/).  A
somewhat related system is used for staging in
[MetaOCaml](http://okmij.org/ftp/ML/MetaOCaml.html).  We obtain a very
high level _macro system_ by combining the two temporal operators `'`
and `~` with Scala 3's `inline` feature. In a nutshell:

 - `inline` copies code from definition site to call site
 - `(')` turns code into syntax trees
 - `(~)` embeds syntax trees in other code.

This approach to macros is very elegant, and has surprising expressive
power. But it might be a little bit too principled. There are still
many bread and butter tasks one cannot do with it. In particular:

 - Syntax trees are "black boxes", we are missing a way to decompose them and analyze their structure and contents.
 - We can only quote and splice expressions, but not other program structures such as definitions or parameters.

We were looking for a long time for ways to augment principled meta
programming by ways to decompose and flexibly reconstruct trees. The
main problem here is choice paralysis - there is basically an infinite
number of ways to expose the underlying structure. Quasi-quotes or
syntax trees? Which constructs should be exposed exactly? What are the
auxiliary types and operations?

If we make some choice here, how do we know that this will be the
right choice for users today? How to guarantee stability of the APIs
in the future? This embarrassment of riches was essentially what
plagued `scala.reflect`. To solve this dilemma, we plan to go
"bottom-up" instead of "top-down". We establish the following
principle:

  _The reflective layer of macros will be isomorphic to Tasty._

This has several benefits:

 - **Completeness**. Tasty is Scala 3's interchange format, so basing the reflection API on it means no information is lost.
 - **Stability**. As an interchange format, Tasty will be kept stable. Its evolution will be carefully managed with a strict versioning system. So the reflection API can be evolved in a controlled way.
 - **Compiler Independence**. Tasty is designed to be independent of the actual Scala compilers supporting it. Besides the Dotty implementation there is now also a proof-of-concept system that shows that `scalac` can generate Tasty trees, and it is even conceivable to generate them from Java. This means that the reflection API can be easily ported to new compilers. If a compiler supports Tasty as the interchange format, it can be made to support the reflection API at the same time.

## Scala in a Nutshell

As a first step towards this goal, we are working on a representation
of Tasty in terms of a suite of compiler-independent data
structures. The [current
status](https://github.com/lampepfl/dotty/blob/master/tests/pos/tasty/definitions.scala)
gives high-level data structures for all aspects of a Tasty file. With
currently about 200 lines of data definitions it reflects every piece of
information that is contained in a Scala program after type
checking. 200 lines is larger than a definition of mini-Lisp, but
much, much smaller than the 30'000 lines or so of a full-blown
compiler frontend!

## Next Steps

The next step, [currently under way](https://github.com/lampepfl/dotty/pull/4279), is to connect these definitions to the Tasty file format. We do this by rewriting them as
[extractors](https://docs.scala-lang.org/tour/extractor-objects.html)
that implement each data type in terms of the data structures used by
the `dotc` compiler which are then pickled and unpickled in the Tasty
file format. An interesting alternative would be to write Tasty
picklers and unpicklers that work directly with reflect trees.

Once this is done, we need to define and implement semantic operations such as

 - what are the members that can be selected on this expression?
 - which subclasses are defined for a sealed trait?
 - does this expression conform to some expected type?

Finally, we need to connect the new lower-level reflection layer to the existing
principled macro system based on quotes and splices. This looks not very difficult. In essence, we
need to define a pair of mappings between high level trees of type
`scala.quoted.Expr[T]` and lower-level Tasty trees of type
`tasty.Term`. Mapping a high-level tree to a low-level one simply
means exposing its structure. Mapping a a low-level tree to a
high-level tree of type `scala.quoted.Expr[T]` means checking that the
low-level tree has indeed the given type `T`. That should be all.

## Future Macros

If this scheme is adopted, it determines to a large degree what Scala 3 macros will
look like. Most importantly, they will run after the typechecking phase is
finished because that is when Tasty trees are generated and
consumed. Running macro-expansion after typechecking has many advantages

 - it is safer and more robust, since the everything is fully typed.
 - it does not affect IDEs, which only run the compiler until typechecking is done.
 - it offers more potential for incremental compilation and parallelization.

But the scheme also restricts the kind of macros that can be expressed:
macros will be blackbox - a macro expansion
cannot influence the type of the expanded expression as seen from the
typechecker. As long as that constraint is satisfied we should be able
to support both `def` macros and annotation macros.

For instance, one could define an annotation macro `@json` that adds a
JSON serializers to a type. The difference with respect to
today's "macro paradise" annotation macros (which are currently not part of the
official Scala distribution) is that in Scala 3 the generated
serializers can be seen only in downstream projects, because the
expansion driven by the annotation happens after type checking.

We believe the lack of whitebox macros can be alleviated to some degree by having
more expressive forms of computed types. A sketch of such as system is outlined
in [Dotty PR 3844](https://github.com/lampepfl/dotty/pull/3844).

The Scala 3 language will also directly incorporate some constructs
that so far required advanced macro code to define. In particular:

- We model lazy implicits directly using
[by-name parameters](http://dotty.epfl.ch/docs/reference/implicit-by-name-parameters.html) instead of through macro.

 - Native [type lambdas](http://dotty.epfl.ch/docs/reference/type-lambdas.html) reduce the need for [kind projector](https://github.com/non/kind-projector).
