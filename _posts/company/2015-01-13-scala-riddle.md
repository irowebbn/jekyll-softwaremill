---
title: Scala type parameters riddle
description: How to implement a method so that no extra type parameters need to be specified?
author: Adam Warski
author_login: warski
categories:
- scala
- company
layout: simple_post
---

**Update 14.01 & 15.01:** new solution at the bottom!

Yesterday we came up with the following Scala riddle.

We have items which may be keyed by integers or strings. We want to implement a `get` method which takes the item's type as a type parameter, and a key of the appopriate type as a value parameter (of course this is a simplification of the real problem :) ):

```scala
// given:
trait Item[K]

class IntItem extends Item[Int]
class StringItem extends Item[String]

// we want the following to compile:
get[IntItem](0)
get[StringItem]("")

// and the following not: 
// get[StringItem](0)
```

A trivial solution is to add another type parameter for the key:

```scala
get[K, T <: Item[K]](id: K)
```

But then we need to repeat the information about the type of the key in the invocation. However, the whole information about the type of the key is present in the type of the item, so why would we need to repeat ourselves?

The first attempt was to add a type member to `Item` which would point to `K`, and which we could use in `get`'s signature:

```scala
trait Item[K] {
  type Key = K
}

def get[T <: Item[_]](k: T#Key) = ???
```

But this fails with an "underscore error":

```scala
// type mismatch;
//  found: Int(0)
//  required: _$1 where type _$1
get[IntItem](0)
```

In other words, the compiler just looks at the `_` we used when defining the constraint on `T` in `get`'s defininition, and is too lazy to actually look inside the type parameter for what's the real value (but that's only my intuition on the compiler's behavior).

How to fix that? [Marcin Kubala](https://github.com/mkubala) came up with two solutions:

First:

```scala
trait Item[K] {
  type Key <: K
}

class IntItem extends Item[Int] {
  override type Key = Int
}

class StringItem extends Item[String] {
  override type Key = String
}

def get[T <: Item[_]](e: T#Key) = ???
```

Second:

```scala
trait ItemLike {
  type Key
}

trait Item[K] extends ItemLike {
  override type Key = K
}

class IntItem extends Item[Int]

class StringItem extends Item[String]
  
def get[T <: ItemLike](e: T#Key) = ???
```

It seems that the compiler only "looks inside" the actual type parameter that was used if the information isn't provided directly in the type that constraints it (the `<: ...` declaration). Let us know if you know the exact mechanism that is in play here.

Or maybe you have a better solution? ;)
 
**Update 14/01 & 15.01:** Following [Paweł Kaczor's](https://twitter.com/newion) remark that the key difference is that `type Key = K` is a type alias, while `type Key <: K` defines an abstract type member, and a later simplification by [Hubert Plociniczak](https://plus.google.com/115793448092501338486/posts) I tried the following, and to my surprise, it works!

```scala
trait Item[K] {
  type Key >: K
}

class IntItem extends Item[Int]
class StringItem extends Item[String]

def get[T <: Item[_]](e: T#Key) = ???
```

Just as in the initial attempt, the crucial difference being that we constrain the type from the bottom (`Key >: K`) instead of defining an alias.

Still, explanations of why the compiler behaves as it does welcome! :)
