Type-Level Programming in Scala, Part 6a: Heterogeneous List Basics
------------------------------------------------------------------------

Posted by Mark on July 6, 2010

An HList is an arbitrary-length tuple. The advantages of an HList over the TupleX classes in the Scala standard library are that we are (in theory) not restricted to a fixed length and we do not need to duplicate methods for each tuple arity. We can (somewhat) easily move between lengths by concatenating tuples or selecting a slice from a tuple. The disadvantages are that we lose the special syntax for tuple construction and types, such as (Int, Boolean) and (3, true), we sometimes run into limitations of the compiler or language, and some useful operations require complicated types.

HLists have been previously demonstrated in Scala (and even in Java). The Scala implementation there used a mix of type members on the HList hierarchy and type class like implicits.

Here we implement four basic operations that let us implement the other operations outside of the HList hierarchy and without a type class for each operation. These basic operations are prepend, fold left, fold right, and a “stuck zipper”. By stuck zipper, I mean a structure that points to a position in the HList and provides the HList before and after that position but cannot move left or right.

With these four operations, we can define:

- Using folds: length, append, reverse, reverse append, last
- Using ‘stuck zipper’: at (select by index), drop, take, replace, remove, map/flatMap at a single index, insert, insert hlist, splitAt

With some extra type-classes, we will also define:

- zip
- heterogenous apply (apply an HList of functions to an HList of inputs)

To start, a very basic HList looks like:

.. code-block:: scala

  sealed trait HList

  final case class HCons[H, T <: HList](head : H, tail : T) extends HList {
     def ::[T](v : T) = HCons(v, this)
  }

  sealed class HNil extends HList {
     def ::[T](v : T) = HCons(v, this)
  }

  // aliases for building HList types and for pattern matching
  object HList {
    type ::[H, T <: HList] = HCons
    val :: = HCons
  }

This basic definition is already useful. For example, the following shows type-safe construction and extraction:

.. code-block:: scala

  // construct an HList similar to the Tuple3 ("str", true, 1.0)
  val x = "str" :: true :: 1.0 :: HNil

  // get the components by calling head/tail
  val s: String = x.head
  val b: Boolean = x.tail.head
  val d: Double = x.tail.tail.head
    // compile error
  //val e = x.tail.tail.tail.head

  // or, decompose with a pattern match

  val f: (String :: Boolean :: Double :: HNil) => String = {
    case "s" :: false :: _ => "test"
    case h :: true :: 1.0 :: HNil => h
      // compilation error because of individual type mismatches and length mismatch
      // case 3 :: "i" :: HNil => "invalid"
    case _ => error("unknown")
  }

Note that we are using :: for HLists, although :+: or another name might be better in practice to distinguish it from the :: used for List concatentation and matching.

Next, we’ll define fold left and right, which we can use to easily implement append, reverse, length, and last.
