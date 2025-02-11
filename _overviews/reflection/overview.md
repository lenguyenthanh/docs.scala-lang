---
layout: multipage-overview
title: Overview

partof: reflection
overview-name: Reflection

num: 1

languages: [ja, zh-cn]
permalink: /overviews/reflection/:title.html
---

<span class="label important" style="float: right;">EXPERIMENTAL</span>

**Heather Miller, Eugene Burmako, Philipp Haller**

*Reflection* is the ability of a program to inspect, and possibly even modify
itself. It has a long history across object-oriented, functional,
and logic programming paradigms.
While some languages are built around reflection as a guiding principle, many
languages progressively evolve their reflection abilities over time.

Reflection involves the ability to **reify** (ie. make explicit) otherwise-implicit
elements of a program. These elements can be either static program elements
like classes, methods, or expressions, or dynamic elements like the current
continuation or execution events such as method invocations and field accesses.
One usually distinguishes between compile-time and runtime reflection depending
on when the reflection process is performed. **Compile-time reflection**
is a powerful way to develop program transformers and generators, while
**runtime reflection** is typically used to adapt the language semantics
or to support very late binding between software components.

Until 2.10, Scala has not had any reflection capabilities of its own. Instead,
one could use part of the Java reflection API, namely that dealing with providing
the ability to dynamically inspect classes and objects and access their members.
However, many Scala-specific elements are unrecoverable under standalone Java reflection,
which only exposes Java elements (no functions, no traits)
and types (no existential, higher-kinded, path-dependent and abstract types).
In addition, Java reflection is also unable to recover runtime type info of Java types
that are generic at compile-time; a restriction that carried through to runtime
reflection on generic types in Scala.

In Scala 2.10, a new reflection library was introduced not only to address the shortcomings
of Java’s runtime reflection on Scala-specific and generic types, but to also
add a more powerful toolkit of general reflective capabilities to Scala. Along
with full-featured runtime reflection for Scala types and generics, Scala 2.10 also
ships with compile-time reflection capabilities, in the form of
[macros]({{site.baseurl }}/overviews/macros/overview.html), as well as the
ability to *reify* Scala expressions into abstract syntax trees.

## Runtime Reflection

What is runtime reflection? Given a type or instance of some object at **runtime**,
reflection is the ability to:

- inspect the type of that object, including generic types,
- to instantiate new objects,
- or to access or invoke members of that object.

Let's jump in and see how to do each of the above with a few examples.

### Examples

#### Inspecting a Runtime Type (Including Generic Types at Runtime)

As with other JVM languages, Scala's types are _erased_ at compile time. This
means that if you were to inspect the runtime type of some instance, that you
might not have access to all type information that the Scala compiler has
available at compile time.

`TypeTag`s can be thought of as objects which carry along all type information
available at compile time, to runtime. Though, it's important to note that
`TypeTag`s are always generated by the compiler. This generation is triggered
whenever an implicit parameter or context bound requiring a `TypeTag` is used.
This means that, typically, one can only obtain a `TypeTag` using implicit
parameters or context bounds.

For example, using context bounds:

    scala> import scala.reflect.runtime.{universe => ru}
    import scala.reflect.runtime.{universe=>ru}

    scala> val l = List(1,2,3)
    l: List[Int] = List(1, 2, 3)

    scala> def getTypeTag[T: ru.TypeTag](obj: T) = ru.typeTag[T]
    getTypeTag: [T](obj: T)(implicit evidence$1: ru.TypeTag[T])ru.TypeTag[T]

    scala> val theType = getTypeTag(l).tpe
    theType: ru.Type = List[Int]

In the above, we first import `scala.reflect.runtime.universe` (it must always
be imported in order to use `TypeTag`s), and we create a `List[Int]` called
`l`. Then, we define a method `getTypeTag` which has a type parameter `T` that
has a context bound (as the REPL shows, this is equivalent to defining an
implicit "evidence" parameter, which causes the compiler to generate a
`TypeTag` for `T`). Finally, we invoke our method with `l` as its parameter,
and call `tpe` which returns the type contained in the `TypeTag`. As we can
see, we get the correct, complete type (including `List`'s concrete type
argument), `List[Int]`.

Once we have obtained the desired `Type` instance, we can inspect it, e.g.:

    scala> val decls = theType.decls.take(10)
    decls: Iterable[ru.Symbol] = List(constructor List, method companion, method isEmpty, method head, method tail, method ::, method :::, method reverse_:::, method mapConserve, method ++)

#### Instantiating a Type at Runtime

Types obtained through reflection can be instantiated by invoking their
constructor using an appropriate "invoker" mirror (mirrors are expanded upon
[below]({{ site.baseurl }}/overviews/reflection/overview.html#mirrors)). Let's
walk through an example using the REPL:

    scala> case class Person(name: String)
    defined class Person

    scala> val m = ru.runtimeMirror(getClass.getClassLoader)
    m: scala.reflect.runtime.universe.Mirror = JavaMirror with ...

In the first step we obtain a mirror `m` which makes all classes and types
available that are loaded by the current classloader, including class
`Person`.

    scala> val classPerson = ru.typeOf[Person].typeSymbol.asClass
    classPerson: scala.reflect.runtime.universe.ClassSymbol = class Person

    scala> val cm = m.reflectClass(classPerson)
    cm: scala.reflect.runtime.universe.ClassMirror = class mirror for Person (bound to null)

The second step involves obtaining a `ClassMirror` for class `Person` using
the `reflectClass` method. The `ClassMirror` provides access to the
constructor of class `Person`. (If this step causes an exception, the easy workaround is to use these flags when starting REPL. `scala -Yrepl-class-based:false`)

    scala> val ctor = ru.typeOf[Person].decl(ru.termNames.CONSTRUCTOR).asMethod
    ctor: scala.reflect.runtime.universe.MethodSymbol = constructor Person

The symbol for `Person`s constructor can be obtained using only the runtime
universe `ru` by looking it up in the declarations of type `Person`.

    scala> val ctorm = cm.reflectConstructor(ctor)
    ctorm: scala.reflect.runtime.universe.MethodMirror = constructor mirror for Person.<init>(name: String): Person (bound to null)

    scala> val p = ctorm("Mike")
    p: Any = Person(Mike)

#### Accessing and Invoking Members of Runtime Types

In general, members of runtime types are accessed using an appropriate
"invoker" mirror (mirrors are expanded upon
[below]({{ site.baseurl}}/overviews/reflection/overview.html#mirrors)).
Let's walk through an example using the REPL:

    scala> case class Purchase(name: String, orderNumber: Int, var shipped: Boolean)
    defined class Purchase

    scala> val p = Purchase("Jeff Lebowski", 23819, false)
    p: Purchase = Purchase(Jeff Lebowski,23819,false)

In this example, we will attempt to get and set the `shipped` field of
`Purchase` `p`, reflectively.

    scala> import scala.reflect.runtime.{universe => ru}
    import scala.reflect.runtime.{universe=>ru}

    scala> val m = ru.runtimeMirror(p.getClass.getClassLoader)
    m: scala.reflect.runtime.universe.Mirror = JavaMirror with ...

As we did in the previous example, we'll begin by obtaining a mirror `m`,
which makes all classes and types available that are loaded by the classloader
that also loaded the class of `p` (`Purchase`), which we need in order to
access member `shipped`.

    scala> val shippingTermSymb = ru.typeOf[Purchase].decl(ru.TermName("shipped")).asTerm
    shippingTermSymb: scala.reflect.runtime.universe.TermSymbol = method shipped

We now look up the declaration of the `shipped` field, which gives us a
`TermSymbol` (a type of `Symbol`). We'll need to use this `Symbol` later to
obtain a mirror that gives us access to the value of this field (for some
instance).

    scala> val im = m.reflect(p)
    im: scala.reflect.runtime.universe.InstanceMirror = instance mirror for Purchase(Jeff Lebowski,23819,false)

    scala> val shippingFieldMirror = im.reflectField(shippingTermSymb)
    shippingFieldMirror: scala.reflect.runtime.universe.FieldMirror = field mirror for Purchase.shipped (bound to Purchase(Jeff Lebowski,23819,false))

In order to access a specific instance's `shipped` member, we need a mirror
for our specific instance, `p`'s instance mirror, `im`. Given our instance
mirror, we can obtain a `FieldMirror` for any `TermSymbol` representing a
field of `p`'s type.

Now that we have a `FieldMirror` for our specific field, we can use methods
`get` and `set` to get/set our specific instance's `shipped` member. Let's
change the status of `shipped` to `true`.

    scala> shippingFieldMirror.get
    res7: Any = false

    scala> shippingFieldMirror.set(true)

    scala> shippingFieldMirror.get
    res9: Any = true

### Runtime Classes in Java vs. Runtime Types in Scala

Those who are comfortable using Java reflection to obtain Java _Class_
instances at runtime might have noticed that, in Scala, we instead obtain
runtime _types_.

The REPL-run below shows a very simple scenario where using Java reflection on
Scala classes might return surprising or incorrect results.

First, we define a base class `E` with an abstract type member `T`, and from
it, we derive two subclasses, `C` and `D`.

    scala> class E {
         |   type T
         |   val x: Option[T] = None
         | }
    defined class E

    scala> class C extends E
    defined class C

    scala> class D extends C
    defined class D

Then, we create an instance of both `C` and `D`, meanwhile making type member
`T` concrete (in both cases, `String`)

    scala> val c = new C { type T = String }
    c: C{type T = String} = $anon$1@7113bc51

    scala> val d = new D { type T = String }
    d: D{type T = String} = $anon$1@46364879

Now, we use methods `getClass` and `isAssignableFrom` from Java Reflection to
obtain an instance of `java.lang.Class` representing the runtime classes of
`c` and `d`, and then we test to see that `d`'s runtime class is a subclass of
`c`'s runtime representation.

    scala> c.getClass.isAssignableFrom(d.getClass)
    res6: Boolean = false

Since above, we saw that `D` extends `C`, this result is a bit surprising. In
performing this simple runtime type check, one would expect the result of the
question "is the class of `d` a subclass of the class of `c`?" to be `true`.
However, as you might've noticed above, when `c` and `d` are instantiated, the
Scala compiler actually creates anonymous subclasses of `C` and `D`,
respectively. This is due to the fact that the Scala compiler must translate
Scala-specific (_i.e.,_ non-Java) language features into some equivalent in
Java bytecode in order to be able to run on the JVM. Thus, the Scala compiler
often creates synthetic classes (i.e. automatically-generated classes) that
are used at runtime in place of user-defined classes. This is quite
commonplace in Scala and can be observed when using Java reflection with a
number of Scala features, _e.g._ closures, type members, type refinements,
local classes, _etc_.

In situations like these, we can instead use Scala reflection to obtain
precise runtime _types_ of these Scala objects. Scala runtime types carry
along all type info from compile-time, avoiding these types mismatches between
compile-time and run-time.

Below, we define a method which uses Scala reflection to get the runtime
types of its arguments, and then checks the subtyping relationship between the
two. If its first argument's type is a subtype of its second argument's type,
it returns `true`.

    scala> import scala.reflect.runtime.{universe => ru}
    import scala.reflect.runtime.{universe=>ru}

    scala> def m[T: ru.TypeTag, S: ru.TypeTag](x: T, y: S): Boolean = {
        |   val leftTag = ru.typeTag[T]
        |   val rightTag = ru.typeTag[S]
        |   leftTag.tpe <:< rightTag.tpe
        | }
    m: [T, S](x: T, y: S)(implicit evidence$1: scala.reflect.runtime.universe.TypeTag[T], implicit evidence$2: scala.reflect.runtime.universe.TypeTag[S])Boolean

    scala> m(d, c)
    res9: Boolean = true

As we can see, we now get the expected result-- `d`'s runtime type is indeed a
subtype of `c`'s runtime type.

## Compile-time Reflection

Scala reflection enables a form of *metaprogramming* which makes it possible
for programs to modify *themselves* at compile time. This compile-time
reflection is realized in the form of macros, which provide the ability to
execute methods that manipulate abstract syntax trees at compile-time.

A particularly interesting aspect of macros is that they are based on the same
API used also for Scala's runtime reflection, provided in package
`scala.reflect.api`. This enables the sharing of generic code between macros
and implementations that utilize runtime reflection.

Note that
[the macros guide]({{ site.baseurl }}/overviews/macros/overview.html)
focuses on macro specifics, whereas this guide focuses on the general aspects
of the reflection API. Many concepts directly apply to macros, though, such
as abstract syntax trees which are discussed in greater detail in the section on
[Symbols, Trees, and Types]({{site.baseurl }}/overviews/reflection/symbols-trees-types.html).

## Environment

All reflection tasks require a proper environment to be set up. This
environment differs based on whether the reflective task is to be done at run
time or at compile time. The distinction between an environment to be used at
run time or compile time is encapsulated in a so-called *universe*. Another
important aspect of the reflective environment is the set of entities that we
have reflective access to. This set of entities is determined by a so-called
*mirror*.

Mirrors not only determine the set of entities that can be accessed
reflectively. They also provide reflective operations to be performed on those
entities. For example, in runtime reflection an *invoker mirror* can be used
to invoke a method or constructor of a class.

### Universes

`Universe` is the entry point to Scala reflection.
A universe provides an interface to all the principal concepts used in
reflection, such as `Types`, `Trees`, and `Annotations`. For more details, see
the section of this guide on
[Universes]({{ site.baseurl}}/overviews/reflection/environment-universes-mirrors.html),
or the
[Universes API docs](https://www.scala-lang.org/api/current/scala-reflect/scala/reflect/api/Universe.html)
in package `scala.reflect.api`.

To use most aspects of Scala reflection, including most code examples provided
in this guide, you need to make sure you import a `Universe` or the members
of a `Universe`. Typically, to use runtime reflection, one can import all
members of `scala.reflect.runtime.universe`, using a wildcard import:

    import scala.reflect.runtime.universe._

### Mirrors

`Mirror`s are a central part of Scala Reflection. All information provided by
reflection is made accessible through these so-called mirrors. Depending on
the type of information to be obtained, or the reflective action to be taken,
different flavors of mirrors must be used.

For more details, see the section of this guide on
[Mirrors]({{ site.baseurl}}/overviews/reflection/environment-universes-mirrors.html),
or the
[Mirrors API docs](https://www.scala-lang.org/api/current/scala-reflect/scala/reflect/api/Mirrors.html)
in package `scala.reflect.api`.
