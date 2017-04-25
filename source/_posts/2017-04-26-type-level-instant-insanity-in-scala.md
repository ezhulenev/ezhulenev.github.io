---
layout: post
title: "Type-Level Instant Insanity in Scala"
date: 2017-04-26 12:15:16 -0500
comments: true
categories: [scala, type level programming]
keywords: scala, type level programming
---

> This post is Scala version of [Haskell Type-Level Instant Insanity by Conrad Parker](http://blog.kfish.org/2007/09/type-level-instant-insanity.html)

This post shows an implementation of Instant Insanity puzzle game at compile time, using powerful Scala type system. This post is
based on amazing article by Conrad Parker in the [Monad Reader Issue 8](https://wiki.haskell.org/wikiupload/d/dd/TMR-Issue8.pdf).
Original article is around 20 pages long, this post is much more concise version of it. Original article is very well written and easy to 
understand, this post should help with jumping from Scala to Haskell code for people who are not familiar with Haskell language.

# Textbook Implementation

*[Instant Insanity](https://en.wikipedia.org/wiki/Instant_Insanity)* puzzle formulated as:

> It consists of four cubes, with faces coloured blue, green, red or white.
> The problem is to arrange the cubes in a vertical pile such that each
> visible column of faces contains four distinct colours.

"Classic" solution in scala can be found [here](https://gist.github.com/ezhulenev/db594992e5f68f435fdc5970e97f02db), this solution
stacks the cubes one at a time, trying each possible orientation of each cube.

I'm going to show how to translate this solution into Scala Type System.

<!-- more -->

# Type-Level Implementation

When I say "Type-Level Implementation", I mean that I'm going to find a solution to the problem without creating any "values",
just operating with types, and I'll do it using Scala compiler only, without running any code.

# ⊥ (Bottom)

I'm working in the type system, so I don't need any value for any of the variables, just the type. In scala there is one
special type, that is subtype of every other type - `Nothing`. There exist no instances of this type, but that's ok because I don't need one.

I'm introducing two functions that will make code look closer to Haskell version, and will save me some typing.

{% codeblock lang:scala %}
def undefined[T]: T = ???
def ⊥[T]: T = undefined
{% endcodeblock %}

{% codeblock lang:scala %}
scala> :type ⊥[Int]
Int

scala> :type ⊥[Seq[String]]
Seq[String]
{% endcodeblock %}


## Simple Types

There are four possible colors. Rather then encoding them as `values` of type `Char`, I'm introducing new types.

{% codeblock lang:scala %}
trait R   // Red
trait G   // Green
trait B   // Blue
trait W   // White
{% endcodeblock %}

{% codeblock lang:scala %}
scala> :type ⊥[R]
R
{% endcodeblock %}

## Parametrized Types

A cube is a thing that can have six faces. In Scala Type System, I use the keyword `trait` to introduce such a thing:

{% codeblock lang:scala %}
trait Cube[u, f, r, b, l, d]
{% endcodeblock %}

I can't get a type of a `Cube` because there is not such thing, `Cube` exist only when it's applied to **type parameters**,
namely *u, f, r, b, l, d*. Applying concrete types to `Cube` will create concrete result type.
One way to think about `Cube` is that it's like a function, but at the type level.

{% codeblock lang:scala %}
scala> :type ⊥[Cube]
<console>:17: error: trait Cube takes type parameters
       ⊥[Cube]
       
scala> :type ⊥[Cube[R, R, R, R, R, R]] // a red cube
Cube[R,R,R,R,R,R]       
{% endcodeblock %}

## Type Aliases

Now I can define the actual cubes in our puzzle as results of applying `Cube` function to concrete types.

{% codeblock lang:scala %}
type CubeRed = Cube[R, R, R, R, R, R]
type CubeBlue = Cube[B, B, B, B, B, B]

type Cube1 = Cube[B, G, W, G, B, R]
type Cube2 = Cube[W, G, B, W, R, R]
type Cube3 = Cube[G, W, R, B, R, R]
type Cube4 = Cube[B, R, G, G, W, W]      
{% endcodeblock %}

## Transformation Functions

Now I need to encode following three functions from textbook solution at a type-level.

{% codeblock lang:scala %}
// Rotate a cube 90 degrees over its Z-axis, leaving up and down in place.
def rot: Cube => Cube = { case Seq(u, f, r, b, l, d) => Seq(u, r, b, l, f, d) }

// Twist a cube around the axis running from the upper-front-right
// corner to the back-left-down corner.
def twist: Cube => Cube = { case Seq(u, f, r, b, l, d) => Seq(f, r, u, l, d, b) }

// Exchange up and down, front and left, back and right.
def flip: Cube => Cube = { case Seq(u, f, r, b, l, d) => Seq(d, l, b, r, f, u) }
{% endcodeblock %}

I'm going to group all of them into single `Transform` trait. Implicit `transform` function can generate an
instance of `Transform` for any type *u, f, r, b, l, d*. I'm not providing any implementation for any of them,
because I never going to *call* them.

{% codeblock lang:scala %}
trait Transform[u, f, r, b, l, d] {
  def rot: Cube[u, f, r, b, l, d] => Cube[u, r, b, l, f, d]
  def twist: Cube[u, f, r, b, l, d] => Cube[f, r, u, l, d, b]
  def flip: Cube[u, f, r, b, l, d] => Cube[d, l, b, r, f, u]
}

implicit def transform[u, f, r, b, l, d] = new Transform[u, f, r, b, l, d] {
  def rot: (Cube[u, f, r, b, l, d]) => Cube[u, r, b, l, f, d] = ⊥
  def twist: (Cube[u, f, r, b, l, d]) => Cube[f, r, u, l, d, b] = ⊥
  def flip: (Cube[u, f, r, b, l, d]) => Cube[d, l, b, r, f, u] = ⊥
}

def rot[u, f, r, b, l, d](cube: Cube[u, f, r, b, l, d])
    (implicit t: Transform[u, f, r, b, l, d]) = t.rot(cube)
def twist[u, f, r, b, l, d](cube: Cube[u, f, r, b, l, d])
    (implicit t: Transform[u, f, r, b, l, d]) = t.twist(cube)
def flip[u, f, r, b, l, d](cube: Cube[u, f, r, b, l, d])
    (implicit t: Transform[u, f, r, b, l, d]) = t.flip(cube)
{% endcodeblock %}

{% codeblock lang:scala %}
scala> :type rot(⊥[Cube1])
Cube[B,W,G,B,G,R]

scala> :type twist(flip(rot(⊥[Cube1])))
Cube[G,B,R,W,B,G]
{% endcodeblock %}

## Type-Level Boolean Algebra

So far we've seen how to construct simple types, and perform type transformations
of one parametrized type into a differently parametrized type.

For solving a puzzle I'll need some rudimentary boolean algebra at a type-level:

- encode true/false in types
- provide boolean `and` operator

 
{% codeblock lang:scala %}
trait True
trait False

trait And[l, r, o]

implicit object ttt extends And[True, True, True]
implicit object tff extends And[True, False, False]
implicit object ftf extends And[False, True, False]
implicit object fff extends And[False, False, False]

def and[l, r, o](l: l, r: r)(implicit and: And[l, r, o]): o = ⊥
{% endcodeblock %} 

{% codeblock lang:scala %}
scala> :type and(⊥[True], ⊥[False])
False

scala> :type and(⊥[True], ⊥[True])
True
{% endcodeblock %} 

## Type-Level Lists

Lists at a type-level can be defined as following atoms:

{% codeblock lang:scala %}
trait Nil
trait :::[x, xs]
{% endcodeblock %}

To make lists useful I'll need concatenate function, that should be also encoded at a type-level.

{% codeblock lang:scala %}
trait ListConcat[l1, l2, l]

implicit def nilConcat[l]: ListConcat[Nil, l, l] = ⊥
implicit def notNilConcat[x, xs, ys, zs](implicit
  lc: ListConcat[xs, ys, zs]
): ListConcat[x ::: xs, ys, x ::: zs] = ⊥

def listConcat[xs, ys, zs](l1: xs, l2: ys)(implicit
  lc: ListConcat[xs, ys, zs]
): zs = ⊥
{% endcodeblock %}

{% codeblock lang:scala %}
scala> :type listConcat(⊥[R ::: Nil], ⊥[G ::: W ::: Nil])
:::[R,:::[G,:::[W,Nil]]]
{% endcodeblock %}

As you can see now I can concatenate two lists at type-level: *[R] concat [G, W]* yields *[R, G, W]*.

## Applyable Type Functions

For this puzzle I need to be able to do things like *flip* each of the cubes in a list, which sounds exactly like *map* at the type level.

First step is abstracting application of a type-level function, so I introduce `Apply` and supported *operations*:

{% codeblock lang:scala %}
trait Rotation
trait Twist
trait Flip

trait Apply[f, a, b]

implicit def apRotation[u, f, r, b, l, d]
  : Apply[Rotation, Cube[u, f, r, b, l, d], Cube[u, r, b, l, f, d]] = ⊥
  
implicit def apTwist[u, f, r, b, l, d]
  : Apply[Twist, Cube[u, f, r, b, l, d], Cube[f, r, u, l, d, b]] = ⊥
  
implicit def apFlip[u, f, r, b, l, d]
  : Apply[Flip, Cube[u, f, r, b, l, d], Cube[d, l, b, r, f, u]] = ⊥

def ap[t, u, f, r, b, l, d, o](r: t, c1: Cube[u, f, r, b, l, d])(implicit
  ap: Apply[t, Cube[u, f, r, b, l, d], o]
): o = ⊥
{% endcodeblock %}

{% codeblock lang:scala %}
scala> :type ap(⊥[Rotation], ⊥[Cube1])
Cube[B,W,G,B,G,R]
{% endcodeblock %}

## Map and Filter

Now I can create a function that recurses over a list and *Applys* another function *f*  to each element. This is type-level
equivalent of the *map* function.

{% codeblock lang:scala %}
trait Map[f, xs, zs]

implicit def mapNil[f]: Map[f, Nil, Nil] = ⊥
implicit def mapCons[f, x, z, xs, zs](implicit
  ap: Apply[f, x, z],
  map: Map[f, xs, zs]
): Map[f, x ::: xs, z ::: zs] = ⊥

def map[f, xs, zs](f: f, xs: xs)(implicit
  map: Map[f, xs, zs]
): zs = ⊥
{% endcodeblock %}  

{% codeblock lang:scala %}
scala> :type map(⊥[Flip], ⊥[Cube1 ::: Cube2 ::: Nil])
:::[Cube[R,B,G,W,G,B],:::[Cube[R,R,W,B,G,W],Nil]]
{% endcodeblock %}

`Filter` function is similar to `Map`:

{% codeblock lang:scala %}
trait Filter[f, xs, zs]

implicit def filterNil[f]: Filter[f, Nil, Nil] = ⊥
implicit def filterCons[f, x, b, xs, ys, zs](implicit
  ap: Apply[f, x, b],
  f: Filter[f, xs, ys],
  apn: AppendIf[b, x, ys, zs]
): Filter[f, x ::: xs, zs] = ⊥
{% endcodeblock %}
 
`Filter` introduced new constraint, `AppendIf`, which takes a boolean values *b*, a value *x* and a list *ys*. 
The given values *x* is appended to *ys* only if *b* is *True*, otherwise *ys* is returned unaltered:

{% codeblock lang:scala %}
trait AppendIf[b, x, ys, zs]

implicit def appendIfTrue[x, ys]: AppendIf[True, x, ys, x ::: ys] = ⊥
implicit def appendIfFalse[x, ys]: AppendIf[False, x, ys, ys] = ⊥

def append[b, x, ys, zs](b: b, x: x, ys: ys)(implicit
  apn: AppendIf[b, x, ys, zs]
): zs = ⊥
{% endcodeblock %}

{% codeblock lang:scala %}
scala> :type append(⊥[True], ⊥[R], ⊥[G ::: W ::: Nil])
:::[R,:::[G,:::[W,Nil]]]

scala> :type append(⊥[False], ⊥[R], ⊥[G ::: W ::: Nil])
:::[G,:::[W,Nil]]
{% endcodeblock %}

## Sequence Comprehensions

Unfortunately sequence comprehensions can't be directly mimiced in Scala Type System, but we can translate the meaning of a given
sequence comprehension using the type-level list functions.

For example, building a list of the possible orientations of a cube involves appending a list of possible
applications of *flip*, so we will need to be able to map over a list and append the original list. [Original sequence comprehension](https://gist.github.com/ezhulenev/db594992e5f68f435fdc5970e97f02db#file-instantinsanity-scala-L18) was:

{% codeblock lang:scala %}
def orientations: Cube => Seq[Cube] = { c: Cube =>
    for {
      ...
      c3 <- Seq(c2, flip(c2))
    } yield c3
  }
{% endcodeblock %}

I create `MapAppend` class in order to compose `Map` and `ListConcat`:

{% codeblock lang:scala %}
trait MapAppend[f, xs, zs]

implicit def mapAppendNil[f]: MapAppend[f, Nil, Nil] = ⊥
implicit def mapAppendCons[f, xs, ys, zs](implicit
  m: Map[f, xs, ys],
  lc: ListConcat[xs, ys, zs]
): MapAppend[f, xs, zs] = ⊥
{% endcodeblock %}

Further, I'll need to be able to do the same twice for *twist* and three times for *rot*:

{% codeblock lang:scala %}
trait MapAppend2[f, xs, zs]

implicit def mapAppend2Nil[f]: MapAppend2[f, Nil, Nil] = ⊥
implicit def mapAppend2Cons[f, xs, ys, _ys, zs](implicit
  m: Map[f, xs, ys],
  ma: MapAppend[f, ys, _ys],
  lc: ListConcat[xs, _ys, zs]
): MapAppend2[f, xs, zs] = ⊥

trait MapAppend3[f, xs, zs]

implicit def mapAppend3Nil[f]: MapAppend3[f, Nil, Nil] = ⊥
implicit def mapAppend3Cons[f, xs, ys, _ys, zs](implicit
  m: Map[f, xs, ys],
  ma2: MapAppend2[f, ys, _ys],
  lc: ListConcat[xs, _ys, zs]
): MapAppend3[f, xs, zs] = ⊥
{% endcodeblock %}

## Orientation

The full sequence comprehension for generating all possible orientations of a cube build upon all
combinations of *rot*, *twist* and *flip*:

{% codeblock lang:scala %}
def orientations: Cube => Seq[Cube] = { c: Cube =>
  for {
    c1 <- Seq(c, rot(c), rot(rot(c)), rot(rot(rot(c))))
    c2 <- Seq(c1, twist(c1), twist(twist(c1)))
    c3 <- Seq(c2, flip(c2))
  } yield c3
}
{% endcodeblock %}

I will implement `Orientation` as an *Apply*able type function. It's defined in terms of applications of
`Rotation`, `Twist` and `Flip`, invoked via various `MapAppend` functions:

{% codeblock lang:scala %}
trait Orientations

implicit def orientations[c, fs, ts, zs](implicit
  ma: MapAppend[Flip, c ::: Nil, fs],
  ma2: MapAppend2[Twist, fs, ts],
  ma3: MapAppend3[Rotation, ts, zs]
): Apply[Orientations, c, zs] = ⊥
{% endcodeblock %}

For any `Cube` this function generates the 24 possible orientations:

{% codeblock lang:scala %}
scala> :type ap(⊥[Orientations], ⊥[Cube1])
:::[Cube[B,G,W,G,B,R],:::[Cube[R,B,G,W,G,B],
:::[Cube[G,W,B,B,R,G],:::[Cube[B,G,R,G,B,W],
:::[Cube[W,B,G,R,G,B],:::[Cube[G,R,B,B,W,G],
:::[Cube[B,W,G,B,G,R],:::[Cube[R,G,W,G,B,B],
:::[Cube[G,B,B,R,W,G],:::[Cube[B,R,G,B,G,W],
:::[Cube[W,G,R,G,B,B],:::[Cube[G,B,B,W,R,G],
:::[Cube[B,G,B,G,W,R],:::[Cube[R,W,G,B,G,B],
:::[Cube[G,B,R,W,B,G],:::[Cube[B,G,B,G,R,W],
:::[Cube[W,R,G,B,G,B],:::[Cube[G,B,W,R,B,G],
:::[Cube[B,B,G,W,G,R],:::[Cube[R,G,B,G,W,B],
:::[Cube[G,R,W,B,B,G],:::[Cube[B,B,G,R,G,W],
:::[Cube[W,G,B,G,R,B],:::[Cube[G,W,R,B,B,G],
Nil]]]]]]]]]]]]]]]]]]]]]]]]
{% endcodeblock %}

## Stacking Cubes

Given two cubes `Cube[u1, f1, r1, b1, l1, d1]` and `Cube[u2, f2, r2, b2, l2, d2]`, I want to check that none of
the corresponding visible faces are the same color: the front sides *f1* and *f2* are not equal, and the right
sides *r1* and *r2* are not equal, and so on.

In order to do this, it's required to define *not equal* relation for all four colors. Given two cubes I can apply
this relations to each pair of visible faces to get four boolean values. To check that all of these are *True*, I will
construct a list of those values and then write generic list function to check if all elements of a list are *True*.

## Not Equal

I'm instantiating `NE` instance for all color combinations.

{% codeblock lang:scala %}
trait NE[x, y, b]

implicit object neRR extends NE[R, R, False]
implicit object neRG extends NE[R, G, True]
implicit object neRB extends NE[R, B, True]
implicit object neRW extends NE[R, W, True]

implicit object neGR extends NE[G, R, True]
implicit object neGG extends NE[G, G, False]
implicit object neGB extends NE[G, B, True]
implicit object neGW extends NE[G, W, True]

implicit object neBR extends NE[B, R, True]
implicit object neBG extends NE[B, G, True]
implicit object neBB extends NE[B, B, False]
implicit object neBW extends NE[B, W, True]

implicit object neWR extends NE[W, R, True]
implicit object neWG extends NE[W, G, True]
implicit object neWB extends NE[W, B, True]
implicit object neWW extends NE[W, W, False]
{% endcodeblock %}

## All

Now, I define a function *all* to check if all elements of a list are *True*.
 
{% codeblock lang:scala %}
trait All[l, b]

implicit def allNil: All[Nil, True] = ⊥
implicit def allFalse[xs]: All[False ::: xs, False] = ⊥
implicit def allTrue[b, xs](implicit all: All[xs, b]): All[True ::: xs, b] = ⊥

def all[b, xs](l: xs)(implicit all: All[xs, b]): b = ⊥
{% endcodeblock %} 

{% codeblock lang:scala %}
scala> :type all(⊥[Nil])
True

scala> :type all(⊥[False ::: Nil])
False

scala> :type all(⊥[True ::: False ::: Nil])
False

scala> :type all(⊥[True ::: True ::: Nil])
True
{% endcodeblock %} 

## Compatible

Now I can write the compatibility check in the Scala Type System, that corresponds original *compatible* funcation:

{% codeblock lang:scala %}
// Compute which faces of a cube are visible when placed in a pile.
def visible: Cube => Seq[Char] = { case Seq(u, f, r, b, l, d) => Seq(f, r, b, l) }

// Two cubes are compatible if they have different colours on every
// visible face.
def compatible: (Cube, Cube) => Boolean = {
  case (c1, c2) => visible(c1).zip(visible(c2)).forall { case (v1, v2) => v1 != v2 }
}
{% endcodeblock %} 

I introduce a new `Compatible` class, it should check that no corresponding visible faces are the same color. It does
validation by evaluating `NE` relationship for each pair of corresponding visible faces. 

{% codeblock lang:scala %}
trait Compatible[c1, c2, b]

implicit def compatibleInstance[f1, f2, bF, r1, r2, bR, b1, b2, bB, l1, l2, bL, u1, u2, d1, d2, b](implicit
  ne1: NE[f1, f2, bF],
  ne2: NE[r1, r2, bR],
  ne3: NE[b1, b2, bB],
  ne4: NE[l1, l2, bL],
  all: All[bF ::: bR ::: bB ::: bL ::: Nil, b]
): Compatible[Cube[u1, f1, r1, b1, l1, d1], Cube[u2, f2, r2, b2, l2, d2], b] = ⊥

def compatible[c1, c2, b](c1: c1, c2: c2)(implicit
  c: Compatible[c1, c2, b]
): b = ⊥
{% endcodeblock %} 

{% codeblock lang:scala %}
scala> :type compatible(⊥[Cube[R, R, R, R, R, R]], ⊥[Cube[B, B, B, B, B, B]])
True

scala> :type compatible(⊥[Cube[R, R, G, R, R, R]], ⊥[Cube[B, B, G, B, B, B]])
False
{% endcodeblock %} 

## Allowed

The above `Compatible` class checks a cube for compatibility with another single cube. In the puzzle,
a cube needs to be compatible with all the other cubes in the pile.

{% codeblock lang:scala %}
// Determine whether a cube can be added to pile of cubes, without
// invalidating the solution.
def allowed: (Cube, Seq[Cube]) => Boolean = {
  case (c, cs) => cs.forall(compatible(_, c))
}
{% endcodeblock %} 

`Allowed` class generalize `Compatible` over list.

{% codeblock lang:scala %}
trait Allowed[c, cs, b]

implicit def allowedNil[c]: Allowed[c, Nil, True] = ⊥
implicit def allowedCons[c, y, b1, ys, b2, b](implicit
  c: Compatible[c, y, b1],
  allowed: Allowed[c, ys, b2],
  and: And[b1, b2, b]
): Allowed[c, y ::: ys, b] = ⊥

def allowed[c, cs, b](c: c, cs: cs)(implicit a: Allowed[c, cs, b]): b = ⊥
{% endcodeblock %} 

{% codeblock lang:scala %}
scala> :type allowed(⊥[CubeRed], ⊥[CubeBlue ::: CubeRed ::: Nil])
False
{% endcodeblock %}

## Solution

Now I'm ready to implement *solutions*:

{% codeblock lang:scala %}
// Return a list of all ways of orienting each cube such that no side of
// the pile has two faces the same.
def solutions: Seq[Cube] => Seq[Seq[Cube]] = {
  case Nil => Seq(Nil)
  case c :: cs => for {
    _cs <- solutions(cs)
    _c <- orientations(c)
    if allowed(_c, _cs)
  } yield _c +: _cs
}
{% endcodeblock %}

I will create a corresponding class `Sultions`, which takes a list of `Cube` as input, and returs a list of
possible solutions, where each solution is a list of `Cube` in allowed orientations.

{% codeblock lang:scala %}
class Solutions[cs, ss]

implicit def solutionsNil: Solutions[Nil, Nil ::: Nil] = ⊥
implicit def solutionsCons[cs, sols, c, os, zs](implicit
  s: Solutions[cs, sols],
  ap: Apply[Orientations, c, os],
  ac: AllowedCombinations[os, sols, zs]
): Solutions[c ::: cs, zs] = ⊥
{% endcodeblock %}

The `AllowedCombinations` class recurses across the solutions so far, checking each against the given orientation.

{% codeblock lang:scala %}
trait AllowedCombinations[os, sols, zs]
implicit def allowedCombinationsNil[os]: AllowedCombinations[os, Nil, Nil] = ⊥
implicit def allowedCombinationsCons[os, sols, as, s, bs, zs](implicit
  ac: AllowedCombinations[os, sols, as],
  mo: MatchingOrientations[os, s, bs],
  lc: ListConcat[as, bs, zs]
): AllowedCombinations[os, s ::: sols, zs] = ⊥
{% endcodeblock %}

Finally `MatchingOrientations` class recurses across the orientations of the new cube,
checking each against a particular solution *sol*.

{% codeblock lang:scala %}
trait MatchingOrientations[os, sols, zs]
implicit def matchingOrientationsNil[sol]: MatchingOrientations[Nil, sol, Nil] = ⊥
implicit def matchingOrientationsCons[os, sol, as, o, b, zs](implicit
  mo: MatchingOrientations[os, sol, as],
  a: Allowed[o, sol, b],
  apn: AppendIf[b, o ::: sol, as, zs]
): MatchingOrientations[o ::: os, sol, zs] = ⊥
{% endcodeblock %}

If orientation is allowed, then the combination *o* is added to the existing solution *sol*, by forming the type *o ::: sol*.

Finally I can solve the puzzle for given cubes:

{% codeblock lang:scala %}
type Cubes = (Cube1 ::: Cube2 ::: Cube3 ::: Cube4 ::: Nil)

def solutions[cs, ss](cs: cs)(implicit sol: Solutions[cs, ss]): ss = ⊥
{% endcodeblock %}

{% codeblock lang:scala %}
scala> :type solutions(⊥[Cubes])
{% endcodeblock %}

For comparison, here is the solution generated by the pure Scala version:

{% coderay %}
[GBWRBG, WGBWRR, RWRBGR, BRGGWW]
[GBRWBG, RRWBGW, RGBRWR, WWGGRB]
[GWRBBG, WBWRGR, RRBGWR, BGGWRW]
[GBBRWG, RGRWBW, RWGBRR, WRWGGB]
[GRBBWG, WWRGBR, RBGWRR, BGWRGW]
[GWBBRG, RBGRWW, RRWGBR, WGRWGB]
[GBBWRG, WRGBWR, RGWRBR, BWRGGW]
[GRWBBG, RWBGRW, RBRWGR, WGGRWB]
{% endcoderay %}

# Conclusion

We've seen how to use Scala Type System as a programming language to solve a given problem, and apparently
it's as powerful as Haskell Type System.

I don't believe that solving this kind of puzzles in a type system makes sense, it took me more then 5 hours
to get a solution, but it shows how expressive it can be.

> Full code for Type-Level Instant Insanity is on [Github](https://gist.github.com/ezhulenev/d741fa7c47d532ec9d1a1bf9aa12fbbc)