Type-Level Programming in Scala, Part 4c: General recursion on Peano numbers
--------------------------------------------------------------------------------

Posted by Mark on June 21, 2010

A core operation that we can use to define arithmetic operations is a fold on the list of integers from N down to _1 (so _0 is treated like an empty list). Using fold right and fold left, we can define operations like addition and multiplication.

We add a FoldR type declaration to Nat as follows:

.. code-block:: scala

  sealed trait Nat {
    type FoldR[Init <: Type, Type, F <: Fold[Nat, Type]] <: Type
  }

where

.. code-block:: scala

  trait Fold[-Elem, Value] {
    type Apply[E <: Elem, V <: Value] <: Value
  }

Fold is defined for types Elem and Value and provides a type function from subtypes of Elem and Value to Value. Compare this with the type of the function that we pass to List.foldRight (represented here as a trait):

.. code-block:: scala

  trait Fold[-Elem, Value] {
    def apply(e: Elem, v: Value): Value
  }

We map a value of type Elem from the data structure and an accumulated value of type Value to a new accumulated value of type Value. For the type level Fold, E and V take the role of the value parameters e and v.

So again, from our definition of FoldR:

.. code-block:: scala

  type FoldR[Init <: Type, Type, F <: Fold[Nat, Type]] <: Type

we see that we accept an initial type Init and that it and all future accumulations will be subtypes of Type. If this isn’t clear, it might help to compare Init to a value and Type to the type of that value in a fold over Lists, for example. F is the Fold subtype that we will use for this right fold. This is the type level function that we provide to FoldR. Again, it is similar to the function you would provide to List.foldRight. Finally, the end result will be a subtype of Type.

As mentioned previously, we can bound our type declaration by one of the type parameters, but the bound cannot change as we recurse. Therfore, Type must be the same throughout recursion. Let’s look at the implementations of FoldR next.

The implementation for _0 is easy. It is just like the implementation for Nil.foldR. We simply return the initial type.

.. code-block:: scala

  sealed trait _0 extends Nat {
    type FoldR[Init <: Type, Type, F <: Fold[Nat, Type]] =
      Init
  }

We can verify that this works:

.. code-block:: scala

  type C = _0#FoldR[ Int, AnyVal, Fold[Nat, AnyVal]]

  // compiles, because C is Int
  implicitly[ C =:= Int ]

  // doesn't compile, because C is not AnyVal
  //implicitly[ C =:= AnyVal ]

The more interesting case is the implementation of Succ:

.. code-block:: scala

  sealed trait Succ[N <: Nat] extends Nat {
    type FoldR[Init <: Type, Type, F <: Fold[Nat, Type]] =
      F#Apply[Succ[N], N#FoldR[Init, Type, F]]
  }

This part recurses on the previous natural number first:

.. code-block:: scala

  N#FoldR[Init, Type, F]

Note that Type is passed along unchanged as required for type-level recursion. The initial type Init and Fold are passed along as well. We have seen that when N is _0, Init will be returned. Therefore, Succ[_0] will apply the Fold F to itself (Succ[_0]) and the initial value Init. Successive Succ[N] will continue applying F#Apply, performing a fold right.

What is interesting is that we can actually preserve types through this and get something useful back out. In the next post in this section, we will look at defining Add, Mult, Exp, and Mod in terms of FoldR.
