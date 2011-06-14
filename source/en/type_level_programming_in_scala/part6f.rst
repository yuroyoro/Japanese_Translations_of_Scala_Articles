Type-Level Programming in Scala, Part 6f: Deriving type class instances through HLists
--------------------------------------------------------------------------------------------------------------------------

Posted by Mark on October 18, 2010

While some parts of this series have not been directly practical (Project Euler #4 in the type system), HLists themselves are definitely practical. The new task engine in sbt 0.9 avoids needing a lot of boilerplate or using a preprocessor while preserving type safety by using HLists (and KLists, a heterogeneous list with a type constructor common to each cell to be discussed in part 8).

For this post, however, let’s look at how we might use HLists to reduce another category of boilerplate. This one is related to defining type class instances, especially ones that are built up for a data type. The examples use scala.Equiv. If you aren’t familiar with Equiv, it is a type class for equality:

.. code-block:: scala

  trait Equiv[T] {
     def equiv(a: T, b: T): Boolean
  }

An instance for Int might look like:

.. code-block:: scala

  implicit def intEquiv: Equiv[Int] = new Equiv[Int] {
     def equiv(a: Int, b: Int) = a == b
  }

For Tuple2, we build on the Equiv instances for its members:

.. code-block:: scala

  implicit def pairEquiv[A,B](implicit a: Equiv[A], b: Equiv[B]): Equiv[ (A,B) ] =
     new Equiv[ (A,B) ] {
        def equiv(t1: (A,B), t2: (A,B)) =
           a.equiv( t1._1, t2._1) && b.equiv( t1._2, t2._2)
     }

We’d need to repeat this for each TupleN. That is a bit annoying, but not too bad. N is fixed to 22 in Scala, so we can generate them ahead of time once and be done with it. However, user classes present an obstacle. We’ll consider case classes in particular, since Scala focuses on generating methods for these. Obviously, users can create as many case classes as they want. You cannot write an Equiv in advance for all of these classes. Instead, for each case class, the user needs to do something like:

.. code-block:: scala

  case class Example[A,B](a: A, b: Seq[B], c: Int)

  implicit def exampleEquiv[A,B](implicit a: Equiv[A], b: Equiv[Seq[B]], c: Equiv[Int]): Equiv[ Example[A,B] ] =
     new Equiv[ Example[A,B]] {
        def equiv(e1: Example[A,B], e2: Example[A,B]) =
           a.equiv( e1.a, e2.a ) &&
           b.equiv( e1.b, e2.b ) &&
           c.equiv( e1.c, e2.c )
     }

This is strictly boilerplate, since we are not saying anything new. We are duplicating the logic for an Equiv for Tuple3. The Example class is basically Tuple3[A,Seq[B], Int] and HLists are capable of representing arbitrary length tuples. We could write functions to convert between a HLists and our case classes. If we then make instances of our type class for HCons and HNil and for any case class that provides a conversion to and from HLists, we can automatically derive type class instances for any case class.

We need help from the compiler to auto-generate the conversion function between a case class and HLists. A prototype of this is here. With this patch to the compiler, the companion object of a case class has implicit conversions to and from the appropriate HList type. For the Example class above, it is roughly equivalent to the following manual definition:

.. code-block:: scala

  case class Example[A,B](a: A, b: Seq[B], c: Int)
  object Example {

     implicit def toHList[A,B](e: Example[A,B]): A :+: Seq[B] :+: Int :+: HNil =
        e.a :+: e.b :+: c :+: HNil

     implicit def fromHList[A,B](hl: A :+: Seq[B] :+: Int :+: HNil): Example[A,B] = {
        val a :+: b :+: c :+: HNil = hl
        Example(a,b,c)
     }
  }

Then, we implement Equiv for HList and for anything convertible to HList:

.. code-block:: scala

  object EquivHList {
     // HNil === HNil
     implicit def equivHNil: Equiv[HNil] = new Equiv[HNil] {
        def equiv(a: HNil, b: HNil) = true
     }
     // given Equiv for the tail and the head,
     //   (h1 :: t1) === (h2 :: t2) when
     //   (h1 === h2) and (t1 === t2)
     implicit def equivHCons[H,T <: HList](implicit equivH: Equiv[H], equivT: Equiv[T]): Equiv[HCons[H,T]] =
        new Equiv[HCons[H,T]] {
           def equiv(a: HCons[H,T], b: HCons[H,T]) =
              equivH.equiv(a.head, b.head) && equivT.equiv(a.tail, b.tail)
        }

     // given:
     //   a type T that is convertible to an HList of type HL
     //   an Equiv instance for HL
     // we can produce an Equiv[T] by mapping T to HL and using Equiv[HL]
     implicit def equivHListable[T, HL <: HList](implicit toHList: T => HL, equivHL: Equiv[HL]): Equiv[T] =
        new Equiv[T] {
           def equiv(a: T, b: T) =
              equivHL.equiv(toHList(a), toHList(b))
        }
  }

So, we need to write something like EquivHList for each type class for which we want to automatically generate instances for case classes. Once we do that, though, we can just do:

.. code-block:: scala

  case class Example[A,B](a: A, b: Seq[B], c: Int)

  object Test {
     assert( Example('a', Seq(1,2,3), 19) === Example('a', Seq(1,2,3), 19) )
     assert( Example('b', Seq(1,2,3), 1) !== Example('a', Seq(1,2,3), 1) )
  }

This example assumes === and !== are provided by an implicit conversion like:

.. code-block:: scala

  implicit def equivEq[A](a1: A)(implicit e: Equiv[A]) = new {
     def ===(a2: A) = e.equiv(a1, a2)
     def !==(a2: A) = !e.equiv(a1, a2)
  }

Predef.conforms also has to be hidden. Otherwise, equivHListable diverges. It would be good to have a nice fix for this.

As a final comment, even without the compiler auto-generating the conversions, it could be less work to manually define the conversions to and from HLists for each case class than it would be to manually implement the type class. This is likely to be the case when you want to implement multiple type classes for a case class.

In the next and last section of this part, I’ll discuss some ways to make working with HLists easier in Scala.
