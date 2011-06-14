Type-Level Programming in Scala, Part 6b: HList folds
-------------------------------------------------------------

Posted by Mark on July 8, 2010

We now build on our basic HList definition by implementing folds. We can then implement functions like append and reverse using these folds. Skip to the end if you want to see the examples first.

Our folds will look very much like the folds for Nat. The basic type-level Foldr signature looks like:

.. code-block:: scala

   type Foldr[Value, F <: Fold[Any, Value], I <: Value] <: Value

Value is the result type and provides the bound that allows us to recurse at the type level. I is the initial type and F is the function used for the fold.

We want to be able to fold values in addition to types. For this, we add an apply method to the Fold trait:

.. code-block:: scala

  trait Fold[-Elem, Value] {
     type Apply[N <: Elem, Acc <: Value] <: Value
     def apply[N <: Elem, Acc <: Value](n: N, acc: Acc): Apply[N, Acc]
  }

HList with Foldr and Foldl types and the corresponding methods foldr and foldl are shown here.

.. code-block:: scala

  sealed trait HList {
     ...

     type Foldr[Value, F <: Fold[Any, Value], I <: Value] <: Value
     def foldr[Value, F <: Fold[Any, Value], I <: Value](f: F, i: I): Foldr[Value, F, I]

     type Foldl[Value, F <: Fold[Any, Value], I <: Value] <: Value
     def foldl[Value, F <: Fold[Any, Value], I <: Value](f: F, i: I): Foldl[Value, F, I]
  }

  final case class HCons[H, T <: HList](head : H, tail : T) extends HList {
     ...

     type Foldr[Value, F <: Fold[Any, Value], I <: Value] =
        F#Apply[H, tail.Foldr[Value, F, I]]

     def foldr[Value, F <: Fold[Any, Value], I <: Value](f: F, i: I): Foldr[Value, F, I] =
        f(head, tail.foldr[Value, F, I](f, i) )

     type Foldl[Value, F <: Fold[Any, Value], I <: Value] =
        tail.Foldl[Value, F, F#Apply[H, I]]

     def foldl[Value, F <: Fold[Any, Value], I <: Value](f: F, i: I): Foldl[Value, F, I] =
        tail.foldl[Value, F, F#Apply[H,I]](f, f(head, i))
  }

  sealed class HNil extends HList {
     ...

     type Foldl[Value, F <: Fold[Any, Value], I <: Value] = I
     def foldl[Value, F <: Fold[Any, Value], I <: Value](f: F, i: I) = i

     type Foldr[Value, F <: Fold[Any, Value], I <: Value] = I
     def foldr[Value, F <: Fold[Any, Value], I <: Value](f: F, i: I) = i
  }

  case object HNil extends HNil

We can define a type-level length function on HList using ‘Inc’ from before:

.. code-block:: scala

   type Length[H <: HList] =
      H#Foldr[Nat, Inc, _0]

   type Inc = Fold[Any, Nat] {
      type Apply[N <: Any, Acc <: Nat] = Succ[Acc]
   }

Using a single fold function that simply prepends the current iteration type to the accumulated HList type,
we can define append, reverse, and reverse append at the type-level:

.. code-block:: scala

   type :::[A <: HList, B <: HList] =
      A#Foldr[HList, AppHCons.type, B]

   type Reverse_:::[A <: HList, B <: HList] =
      A#Foldl[HList, AppHCons.type, B]

   type Reverse[A <: HList] =
      A#Foldl[HList, AppHCons.type, HNil]

   object AppHCons extends Fold[Any, HList] {
      type Apply[N <: Any, H <: HList] = N :: H

      // used later for value-level implementations
      def apply[A,B <: HList](a: A, b: B) = HCons(a, b)
   }

We define a type-class that will provide the value-level operations:

.. code-block:: scala

  sealed trait HListOps[B <: HList] {
     def length: Int
     def :::[A <: HList](a: A): A ::: B
     def reverse: Reverse[B]
     def reverse_:::[A <: HList](a: A): A Reverse_::: B
  }

and implement it with folds that are straightforward translations of our type-level folds:

.. code-block:: scala

   implicit def hlistOps[B <: HList](b: B): HListOps[B] =
      new HListOps[B] {

         def length =
            b.foldr(Length, 0)

         def reverse =
            b.foldl[HList, AppHCons.type, HNil](AppHCons, HNil)

         def :::[A <: HList](a: A): A#Foldr[HList, AppHCons.type, B] =
            a.foldr[HList, AppHCons.type, B](AppHCons, b)

         def reverse_:::[A <: HList](a: A): A Reverse_::: B =
            a.foldl[HList, AppHCons.type, B](AppHCons, b)
      }

   object Length extends Fold[Any, Int] {
      type Apply[N <: Any, Acc <: Int] = Int
      def apply[A,B <: Int](a: A, b: B) = b+1
   }

Some examples of using these:

.. code-block:: scala

   // construct a heterogeneous list of length 3 and type
   //  Int :: String :: List[Char] :: HNil
   val a = 3 :: "ai4" :: List('r','H') :: HNil

   // construct a heterogeneous list of length 4 and type
   //  Char :: Int :: Char :: String :: HNil
   val b = '3' :: 2 :: 'j' :: "sdfh" :: HNil

      // append the two HLists
   val ab = a ::: b
      // check the types by assigning to an explicitly annotated val
   val checkAB : Int :: String :: List[Char] :: Char :: Int :: Char :: String :: HNil = ab
      // check the values by matching literal patterns
   val 3 :: "ai4" :: List('r','H') :: '3' :: 2 :: 'j' :: "sdfh" :: HNil = ab

      // length of an HList, on both type and value level
   val 7 = ab.length
   implicitly[_7 =:= ab.Length]

      // reverse
   val reversed = b.reverse
      // check types
   val checkReversed : String :: Char :: Int :: Char :: HNil = reversed
      // check values
   val "sdfh" :: 'j' :: 2 :: '3'  :: HNil = reversed

      // last (implementation not shown in this post- it is implemented with a foldl)
   val last = reversed.last
      // check type
   val checkLast : Char = last
      // check value
   val '3' = last

      // reverse_:::
   val reverseAppend = a reverse_::: b
      // check types
   val checkReverseAppend : List[Char] :: String :: Int :: Char :: Int :: Char :: String :: HNil = reverseAppend
      // check values
   val  List('r','H') :: "ai4" :: 3 :: '3' :: 2 :: 'j' :: "sdfh" :: HNil = reverseAppend


For comparison, see the MetaScala and the Haskell implementations.
