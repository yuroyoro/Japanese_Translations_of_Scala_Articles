Type-Level Programming in Scala, Part 4a: Peano number basics
--------------------------------------------------------------------------

Posted by Mark on June 16, 2010

As mentioned in the opening post, we can do recursion at the type-level in Scala. The first application for this will be representing numbers in the type system (Peano numbers). One use of these is type-safe indexing into HLists, which we will do in a later post. The basic idea is not new, of course, but we are still building up the tools we need for later posts.

See also:
http://www.haskell.org/haskellwiki/Peano_numbers
http://trac.assembla.com/metascala/browser/src/metascala/Nats.scala

A basic representation of non-negative integers at the type level looks like:

.. code-block:: scala

  sealed trait Nat
  sealed trait _0 extends Nat
  sealed trait Succ[N <: Nat] extends Nat

We can construct larger integers with Succ:

.. code-block:: scala

  type _1 = Succ[_0]
  type _2 = Succ[_1]
  ...

Like for Bool, we add some structure to Nat. There are three core operations that we will define, all at the type level.

The operation required for indexing into HLists is similar to the If we defined on Bool. Let’s call it Match. The definition is:

.. code-block:: scala

  sealed trait Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] <: Up
  }
  sealed trait _0 extends Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] = IfZero
  }
  sealed trait Succ[N <: Nat] extends Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] = NonZero[N]
  }

It is similar to our Bool#If except that we accept a type constructor for the case when the Nat is Succ. That is, if Match is called on Succ[N], the result is NonZero[N]. If the actual Nat type is _0, IfZero is returned. The Up parameter defines a bound for the result, as for Bool#If.

We can see that this allows us to deconstruct a Nat. For example, we can define a type declaration that determines if its parameter is _0:

.. code-block:: scala

  type Is0[A <: Nat] = A#Match[ConstFalse, True, Bool]
  type ConstFalse[A] = False

If A is _0, True will be the result of A#Match. If A is Succ[N], ConstFalse[N] is the result. ConstFalse always evaluates to False, so the result is False for Succ.

Match is the only operation that we need for HLists. We’ll continue by defining comparisons, addition, multiplication, modulus, and exponentiation just for fun anyway.
