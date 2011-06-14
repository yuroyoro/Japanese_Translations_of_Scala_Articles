Type-Level Programming in Scala, Part 6c: HList Indexing
-------------------------------------------------------------

Posted by Mark on July 12, 2010

Next, we implement some indexing operations on HLists. We’d like to drop or take n elements, get the element at index i, split into two HLists around index i, and so forth.

The idea is to take an HList and Nat and produce a zipper-like structure Indexed. This structure provides the value and type at the index given by the Nat. It also provides the HList before and after the index.

Indexed is a stuck zipper, however. It cannot move left or right, but is fixed at the initial index. (Unfortunately, even this stuck zipper can be problematic for the compiler.)

The basic Indexed interface looks like:

.. code-block:: scala

  sealed trait Indexed {
     type Before <: HList
     type After <: HList
     type At
     def withIndex[R](f: (Before, At, After) =>  R): R
  }

As an example, the Indexed instance for the Nat _2 and the HList 3 :: true :: "blue" :: 'h' :: HNil is:

.. code-block:: scala

  new Indexed {
     type Before = Int :: Boolean :: HNil
     type After = Char :: HNil
     type At = String
     def withIndex[R](f: (Before, At, After) =>  R): R =
       f( 3 :: true :: HNil, "blue", 'h' :: HNil )
  }

Before we look at creating an instance of Indexed for an arbitrary Nat and HList, let’s see that we can implement several indexing operations in terms of withIndex:

.. code-block:: scala

  sealed trait Indexed {
     ...
     // get the element at this index
     def at =
        withIndex( (_, a, _) => a)

     // drop all elements before this index
     def drop =
        withIndex( (_, a, t) => a :: t)

     // take all elements before this index
     def take =
        withIndex( (b, _, _) => b)

     // replace the value at this index with 'a'
     def replace[A](a: A) =
        withIndex( (b, _, t) => b ::: a :: t )

     // remove the value at this index
     def remove =
        withIndex( (b, _, t) => b ::: t )

     // update the value at this index by applying 'f' to the current value
     def map[B](f: At => B) =
        withIndex( (b, a, t) => b ::: f(a) :: t )

     // remove the value at this index, apply 'f', and insert the resulting HList
     def flatMap[B <: HList](f: At => B) =
        withIndex( (b, a, t) => b ::: f(a) ::: t )

     // insert the given value at this index
     def insert[C](c: C) =
        withIndex( (b, a, t) => b ::: c :: a :: t)

     // insert the given HList at this index
     def insertH[C <: HList](c: C) =
        withIndex( (b, a, t) => b ::: c ::: a :: t )

     // return the HList before this index and the HList starting at this index
     def splitAt =
        withIndex( (b, a, t) => (b, a :: t))
  }

The implementation is split into two cases. The first is Indexed0, which represents the location of the pointer. It points to the head of an HCons cell. The tail of the cell is After and Before is empty.

.. code-block:: scala

  final class Indexed0[H, T <: HList](val hc: H :: T) extends Indexed {
     type Before = HNil
     type After = T
     type At = H
     def withIndex[R](f: (Before, At, After) =>  R): R = f(HNil, hc.head, hc.tail)
  }

IndexedN builds up the Before HList. It prepends an element to another Indexed’s Before and delegates At and After to that Indexed. Ultimately, the terminating Indexed must be an Indexed0.

.. code-block:: scala

  final class IndexedN[H, I <: Indexed](h: H, iTail: I) extends Indexed {
     type Before = H :: I#Before
     type At = I#At
     type After = I#After
     def withIndex[R](f: (Before, At, After) =>  R): R = iTail.withIndex( (before, at, after) => f( HCons(h, before), at, after) )
  }

Now we need to actually get an Indexed for an HList. To do this, we define a type member toI[N <: Nat] on HList. This type member defines the Indexed type for that HList and Nat. We can then use implicits to provide this type.

.. code-block:: scala

  sealed trait HList {
     ...
     type toI[N <: Nat] <: Indexed
  }

  final case class HCons[H, T <: HList](head : H, tail : T) extends HList {
     ...
     // match on N
     //   If it is _0, the Indexed type should point to this cell (so it is Indexed0)
     //   otherwise, the index is to the right, so recurse on N-1 and return an IndexedN
     type toI[N <: Nat] = N#Match[ IN, Indexed0[H, T], Indexed]
     type IN[M <: Nat] = IndexedN[H, tail.toI[M]]
  }
  sealed class HNil extends HList {
     ...
     type toI[N <: Nat] = Nothing
  }

So, given an HList h and an Nat index N, we know that the Indexed type we want is h.toI[N]. We want a function h => h.toI[N]. We can do this with implicits as follows.

.. code-block:: scala

   // defined on HListOps type class, where HL is the type of the HList
   def i[N <: Nat](implicit in: HL => toI[N]) = in(this)

  object Indexed {
     implicit def indexed0[H, T <: HList](hc: H :: T): Indexed0[H, T] =
        new Indexed0[H, T](hc)

     implicit def indexedN[H, T <: HList, I <: Indexed](hc: H :: T)(implicit iTail: T => I): IndexedN[H, I] =
        new IndexedN[H, I](hc.head, iTail(hc.tail))
  }

Examples:

.. code-block:: scala

  val x = 3 :: true :: "asfd" :: false :: 'k' :: () :: 13 :: 9.3 ::  HNil

  // get the boolean value 'true' at index 3
     /* false */
  val b2: Boolean = x.i[_3].at

  // drop everything before index 3 and then get the first value
     /* false */
  val pre: Boolean = x.i[_3].drop.i[_0].at

  // replace the boolean value at index 3 with the integer 19
     /* 3 :: true :: asfd :: 19 :: k :: () :: 13 :: 9.3 :: HNil */
  val rep = x.i[_3].replace(19)

  // replace the character at index 4 with true if it is lowercase, false otherwise
     /* 3 :: true :: asfd :: false :: true :: () :: 13 :: 9.3 :: HNil */
  val mp = x.i[_4].map(_.isLower)

  // remove the value at index 5
     /* 3 :: true :: asfd :: false :: k :: 13 :: 9.3 :: HNil */
  val rm = x.i[_5].remove

  // remove the value at index 2 and insert an HList derived from its value
     /* 3 :: true :: a :: sfd :: false :: k :: () :: 13 :: 9.3 :: HNil */
  val fmp = x.i[_2].flatMap( s => s.charAt(0) :: s.substring(1) :: HNil )

  // insert a value at the beginning of the HList
     /* List(3, 4) :: 3 :: true :: asfd :: false :: k :: () :: 13 :: 9.3 :: HNil */
  val ins0 = x.i[_0].insert(List(3,4))

  // insert a value at index 7
     /* 3 :: true :: asfd :: false :: k :: () :: 13 :: -3.0 :: 9.3 :: HNil */
  val ins7 = x.i[_7].insert(-3.0f)

  // insert an HList at index 3
     /* 3 :: true :: asfd :: h :: false :: Some(3) :: None :: false :: k :: 13 :: 9.3 :: HNil */
  val insH = rm.i[_3].insertH( 'h' :: b2 :: Some(3) :: None :: HNil )

  // split the HList around index 6
     /* (3 :: true :: asfd :: false :: k :: () :: HNil, 13 :: -3.0 :: 9.3 :: HNil) */
  val (aa, bb) = ins7.i[_6].splitAt

  // encoding of drop right
     /* 3 :: true :: asfd :: false :: k :: HNil */
  val dropRight = x.reverse.i[_3].drop.reverse

Next we will see how to define zip and unzip to combine and separate HLists of tuples.
