Type-Level Programming in Scala, Part 6e: HList Apply
-------------------------------------------------------------

Posted by Mark on October 15, 2010

We’ll continue by defining happly, a heterogeneous apply method. It takes an HList of functions and applies them to the corresponding values in an input HList, producing an HList of results.

First, the example usage looks like:

.. code-block:: scala

  // data HLists of type  Double :: Char :: HNil
  val y1 = 9.75 :: 'x' :: HNil
  val y2 = -2.125 :: 'X' :: HNil

  // The functions to apply.  z has type:
  //
  //   (Double => Double) :: (Char => Boolean) :: HNil
  //
  val z = ((_: Double) + .5) :: ( (_: Char).isUpper) :: HNil

  // apply to first input HList y1
  val z1 = happly(z)(y1)

  // check types
  val z1Types : Double :: Boolean :: HNil = z1

  // check values
  val 10.25 :: false :: HNil = z1

  // apply to second input HList y2
  val z2 = happly(z)(y2)

  // check types
  val z2Types : Double :: Boolean :: HNil = z2

  // check values
  val -1.625 :: true :: HNil = z2

We’ll implement happly using a type class HApply, which is essentially Function1. We can’t actually use Function1 because existing implicits related to Function1 get in the way.

.. code-block:: scala

  sealed trait HApply[-In, +Out] {
    def apply(in: In): Out
  }

The idea is that given an HList of functions, we produce an HApply that accepts an HList of parameters of the right type and produces an HList of results. For the function 'z' from the example, we want an HApply of type:

.. code-block:: scala

  HApply[ (Double :: Char :: HNil), (Double :: Boolean :: HNil)]

There are two basic cases for this: HNil and HCons. The easy case is mapping an HNil to an HNil, handled by happlyNil.

.. code-block:: scala

   implicit def happlyNil(h: HNil) : HApply[HNil, HNil] =
      new HApply[HNil, HNil] {
         def apply(in: HNil) = HNil
      }

As usual, HCons is the interesting case. We accept an HCons cell with a head that is a function and produce an HApply that will accept an input HCons cell of the right type. The HApply then applies the head function to the head value and recurses on the tail. The HApply to use for recursion is provided as an implicit and this is how we require that one HList is an HList entirely of functions and the other HList contains values of the right type to be provided as inputs to those functions.

.. code-block:: scala

   implicit def happlyCons[InH, OutH, TF <: HList, TIn <: HList, TOut <: HList]
      (implicit applyTail: TF => HApply[TIn, TOut]) =
         (h: HApply[InH :: TIn, OutH :: TOut]) =>

      new HApply[InH :: TIn, OutH :: TOut] {
         def apply(in: InH :: TIn) =
            HCons( h.head(in.head), applyTail(h.tail)(in.tail) )
      }

In the example, we have:

.. code-block:: scala

   val y1 = 9.75 :: 'x' :: HNil

   val z: (Double => Double) :: (Char => Boolean) :: HNil =
       ((_: Double) + .5) :: ( (_: Char).isUpper) :: HNil

So, for happly(z)(y1), our implicit is constructed with:

.. code-block:: scala

   happlyCons[ Double, Double, Char => Boolean :: HNil, Char :: HNil, Boolean :: HNil](
      happlyCons[ Char, Boolean, HNil, HNil, HNil](
         happlyNil
      )
   )
The first applyCons constructs an HApply that uses the head of z, a function of type 'Double => Double', to map an HList with a head of type Double to an HList with a head of type Double. It uses the HApply from the second happlyCons for mapping the tail of the input HList.

This second HApply uses the second element of z, a function of type 'Char => Boolean', to map an HList with a head of type Char to an HList with a head of type Boolean. Because this is the last element, the recursion ends with happlyNil mapping HNil to HNil.

Finally, we define an entry point. Given an HList of functions and an HList of arguments to those functions, we use happly to grab an HApply implicitly and produce the resulting HList with it:

.. code-block:: scala

   def happly[H <: HList, In <: HList, Out <: HList]
      (h: H)(in: In)(implicit toApply: H => HApply[In, Out]): Out =
         toApply(h)(in)
