Type-Level Programming in Scala, Part 6g: Type-indexed HLists
----------------------------------------------------------------------

Posted by Mark on October 22, 2010

Thanks to Ross Mellgren for the impetus for adding another post to the HList part.

In 6c, we performed a variety of operations based on a position in an HList that was selected by a type-level natural number (Nat). In this post, we select the position by type. Bug #3201 makes the final syntax a bit more explicit than we’d like and bug #2741, which also affected 6c, means separate compilation crashes the compiler.

Similar to 6c, we take an HList and a type and produce an Indexed instance, which provides the value and type at a selected index and the HList before and after the index. We can reuse the implementation of Indexed by defining a few new implicits.

The original approach used a type-level function to compute the necessary Indexed type completely within the type-level:

.. code-block:: scala

  trait HList {
     type toI[N <: Nat] <: Indexed
  }

For an HList type HL and a Nat N, toI gave us the Indexed type for the position selected by the number N. We could then require an implicit Indexed parameter of type HL#toI[N] and define implicits that built up requested Indexed values.

This won’t work here, however. We want to select the position in the HList where the cell type H is the same as the requested type S. We cannot compute the Indexed type using a type-level-only solution because there is no general type-level equality function. We need to rely on implicit evidence of equality of two types using =:=. A bit more concretely, you can’t write:

.. code-block:: scala

   type TypeEq[A, B] <: Bool

but you can require evidence (as a value) that A and B are equal:

.. code-block:: scala

   def typeEq[A,B](implicit ev: A =:= B) ...

Remember from 6c that Indexed is comprised of two pieces: Indexed0, which represents the location of the pointer into the HList, and IndexedN, which represents locations before the pointer. In order to select the position by type, we will use a wrapper class that records the type S selected and the mapping from HList to Indexed:

.. code-block:: scala

   sealed trait Tip[S, HL <: HList, Ind <: Indexed] {
      def apply(hl: HL): Ind
   }

So, an instance of Tip for an HList HL and a type S can provide an Indexed instance on which we can call methods like ‘at’, ‘remove’, or ‘map’.

The actual work is constructing Tip instances. There is one case for Indexed0 and one for IndexedN. When we have evidence that the type of the head of an HCons cell is the same as the type we are looking for, we provide an Indexed0 value that points to this cell. Again, we wrap it in Tip in order to mark that we have selected a cell of type S and to provide a function that maps from the HCons cell to the Indexed instance that we ultimately want.

.. code-block:: scala

   implicit def tindexed0[S, H, T <: HList](implicit ev: S =:= H): Tip[S, H :: T, Indexed0[H, T]] =
      new Tip[S, H :: T, Indexed0[H,T]] {
         def apply(hc: H :: T) = new Indexed0[H, T](hc)
      }

Indexed0 points to the head of the selected HCons cell and references its tail. We then need to build up IndexedN instances from the terminating Indexed0 to reference the cells before the selected one. For an HCons cell of type H :: T, we require that the type S has been found in the tail T. That is, we need a Tip instance for the tail type T, searched type S, and some Indexed instance I. From this we can provide a Tip for the current cell.

.. code-block:: scala

   implicit def tindexedN[H, T <: HList, I <: Indexed, S](implicit iTail: Tip[S, T, I] ): Tip[S, H :: T, IndexedN[H, I]] =
      new Tip[S, H :: T, IndexedN[H, I]] {
         def apply(hc: H :: T) = new IndexedN[H, I](hc.head, iTail(hc.tail))
      }

Note that this will result in ambiguous implicits when there are multiple cells with the same type.

To connect the above to an HList for a requested type, we add a method ‘t’ to HCons:

.. code-block:: scala

   def t[S]: TipDummy[S, H :: T]

   sealed class TipDummy[S, HL <: HList](val hl: HL)

where H :: T is the type of the HCons cell. We then provide an implicit conversion from TipDummy to Indexed:

.. code-block:: scala

   implicit def tipToInd[S, HL <: HList, I <: Indexed](dummy: TipDummy[S, HL])(implicit tip: Tip[S, HL, I]): I = tip(dummy.hl)

The intermediate TipDummy accomplishes partial type parameter application. We want to be able to explicitly specify the type to select without having to provide the Indexed and HList types. We want those to be inferred. In practice, we need to explicitly call the tipToInd conversion because of Scala bug #3201. So, instead of something like:

.. code-block:: scala

  hlist.t[Boolean].at

we have to do:

.. code-block:: scala

  tipToInd(hlist.t[Boolean]).at

Examples (explicit calls to tipToInd are optimistically omitted):

.. code-block:: scala

   val x = 3 :: true :: "asfd" :: 'k' :: () :: 9.3 ::  HNil

   // get the Boolean value
      /* true */
   val b2: Boolean = x.t[Boolean].at

   // drop everything before the String
      /* asfd :: k :: () :: 9.3 :: HNil */
   val pre = x.t[String].drop

   // replace the String value with the integer 19
      /* 3 :: true :: 19 :: k :: () :: 9.3 :: HNil */
   val rep = x.t[String].replace(19)

   // replace the Char with true if it is lowercase, false otherwise
      /* 3 :: true :: asfd :: true :: () :: 9.3 :: HNil */
   val mp = x.t[Char].map(_.isLower)

   // remove the Unit value
      /* 3 :: true :: asfd :: k :: 9.3 :: HNil */
   val rm = x.t[Unit].remove

   // remove the String and insert an HList derived from its value
      /* 3 :: true :: a :: sfd :: k :: () :: 9.3 :: HNil */
   val fmp = x.t[String].flatMap( s => s.charAt(0) :: s.substring(1) :: HNil )

   // insert a value before the Int
      /* List(3, 4) :: 3 :: true :: asfd :: k :: () :: 9.3 :: HNil */
   val ins0 = x.t[Int].insert(List(3,4))

   // insert a value before the Double
      /* 3 :: true :: asfd :: false :: k :: () :: -3.0 :: 9.3 :: HNil */
   val ins7 = x.t[Double].insert(-3.0f)

   // insert an HList before the String
      /* 3 :: true :: h :: true :: Some(3) :: None :: asfd :: k :: 9.3 :: HNil */
   val insH = rm.t[String].insertH( 'h' :: b2 :: Some(3) :: None :: HNil )

   // split the HList around the Unit value
      /* (3 :: true :: asfd :: k :: HNil, () :: -3.0 :: 9.3 :: HNil) */
   val (aa, bb) = ins7.t[Unit].splitAt

   // encoding of drop right
      /* 3 :: true :: asfd :: k :: HNil */
   val dropRight = x.reverse.t[Char].drop.reverse

