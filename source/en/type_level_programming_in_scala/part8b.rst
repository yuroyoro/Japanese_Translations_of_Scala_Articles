Type-Level Programming in Scala, Part 8b: KList basics
-------------------------------------------------------------

Posted by Mark on November 3, 2010

In the absence of rank-2 types, it can be useful to have a higher-kinded heterogeneous list, which I’ll call KList here. A KList defines a type constructor M[_] that is used to construct the type for all cells in the list. The parameter passed to this type constructor can be different for each cell, which is the heterogeneous part. One use of a KList is to define a generic zipWith function. KLists are also used in the implementation of the new task engine in sbt 0.9. Each of these applications will be described in subsequent posts.

We’ll start with the basic definition of KList, which looks like:

.. code-block:: scala

  sealed trait KList[+M[_], HL <: HList]

  final case class KCons[H, T <: HList, +M[_]](head: M[H], tail: KList[M,T]) extends KList[M, H :+: T] {
    // prepend
    def :^: [N[X] >: M[X], G](g: N[G]) = KCons(g, this)
  }

  sealed class KNil extends KList[Nothing, HNil] {
    def :^: [M[_], H](h: M[H]) = KCons(h, this)
  }

  object KNil extends KNil

  object KList {
    // nicer alias for pattern matching
    val :^: = KCons
  }

It looks similar to HList with the exception of the type constructor M. We keep the type of ‘head’ in KCons in two pieces: the type constructor M common to all cells in the KList and the type parameter to M that is specific to this cell. The full type of ‘head’ is then M[H].

An example construction:

.. code-block:: scala

  val m = List(1, 2, 3, 4) :^: List("str1", "str2") :^: KNil

This has type:

.. code-block:: scala

  KCons[Int,java.lang.String :: HNil,List]

Note that we can mix type constructors that are compatible:

.. code-block:: scala

  val m = Seq(1, 2, 3, 4) :^: List("str1", "str2") :^: KNil
  m: KCons[Int,java.lang.String :: HNil,Seq]

Ones that are not compatible crash the compiler:

.. code-block:: scala

  val m = Seq(1, 2, 3, 4) :^: Option("str1", "str2") :^: KNil
  scala.tools.nsc.symtab.Types$NoCommonType: lub/glb of incompatible types: [X]Option[X] and Seq

It is not possible to have types inferred in several cases, such as when the type constructor is Id, where type Id[X] = X :

.. code-block:: scala

  // does not compile
  val p = 1 :^: true :^: KNil

A key use of a KList is to apply a natural transformation to its contents. We have kept the type constructor separate from the type parameters, which means we can apply a natural transformation M ~> N to each element and preserve the underlying type parameters. As an example, consider our heterogeneous list of Lists:

.. code-block:: scala

  val m = List(1, 2, 3, 4) :^: List("str1", "str2") :^: KNil

and a natural transformation that takes a List and calls headOption on it:

.. code-block:: scala

  val headOption = new (List ~> Option) {
    def apply[T](list: List[T]): Option[T] =
      list.headOption
  }

Then, apply headOption to m:

.. code-block:: scala

  val heads = m transform headOption
  heads: KCons[Int,(java.lang.String :: HNil),Option] = KCons(Some(1),KCons(Some(str1),KNil))

We get a KList of Options, preserving the knowledge that the first element has type Option[Int] and the second has type Option[String].

The ‘transform’ method on KList is straightforward to implement:

.. code-block:: scala

  sealed trait KList[+M[_], HL <: HList] {
    ...
    def transform[N[_]](f: M ~> N): KList[N, HL]
  }

  final case class KCons[H, T <: HList, +M[_]](head: M[H], tail: KList[M,T]) extends KList[M, H :+: T] {
    ...
    def transform[N[_]](f: M ~> N) = KCons( f(head), tail transform f )
  }

  sealed class KNil extends KList[Nothing, HNil] {
    ...
    def transform[N[_]](f: Nothing ~> N) = KNil
  }

We can add another method that down-converts a KList to its underlying HList. For example, we might reduce each List in our KList above to its head element:

.. code-block:: scala

  val head = new (List ~> Id) {
    def apply[T](list: List[T]): T =
      list.head
  }

  val heads = m down head
  heads: Int :: java.lang.String :: HNil = 1 :: str1 :: HNil

The definition of ‘down’ looks like:

.. code-block:: scala

  sealed trait KList[+M[_], HL <: HList] {
    ...
    // For converting KList to an HList
    def down(f: M ~> Id): HL
  }

  final case class KCons[H, T <: HList, +M[_]](head: M[H], tail: KList[M,T]) extends KList[M, H :+: T] {
    ...
    def down(f: M ~> Id) = HCons(f(head), tail down f)
  }

  sealed class KNil extends KList[Nothing, HNil] {
    ...
    def down(f: Nothing ~> Id) = HNil
  }

We will use ‘down’ and ‘transform’ in the next section to implement zipWith for arbitrary arity.
