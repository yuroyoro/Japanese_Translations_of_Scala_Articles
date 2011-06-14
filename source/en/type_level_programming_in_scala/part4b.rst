Type-Level Programming in Scala, Part 4b: Comparing Peano numbers
-----------------------------------------------------------------------------

Posted by Mark on June 17, 2010

Next, let’s look at comparing natural numbers. Define a Comparison type as follows:

.. code-block:: scala

  sealed trait Comparison {
    type Match[IfLT <: Up, IfEQ <: Up, IfGT <: Up, Up] <: Up

    type gt = Match[False, False, True, Bool]
    type ge = Match[False, True, True, Bool]
    type eq = Match[False, True, False, Bool]
    type le = Match[True, True, False, Bool]
    type lt = Match[True, False, False, Bool]
  }

The implementations look like:

.. code-block:: scala

  sealed trait GT extends Comparison {
    type Match[IfLT <: Up, IfEQ <: Up, IfGT <: Up, Up] = IfGT
  }
  sealed trait LT extends Comparison {
    type Match[IfLT <: Up, IfEQ <: Up, IfGT <: Up, Up] = IfLT
  }
  sealed trait EQ extends Comparison {
    type Match[IfLT <: Up, IfEQ <: Up, IfGT <: Up, Up] = IfEQ
  }

We can then define a Compare type member for Nat. It is shown here along with the Match type we defined in the last post.

.. code-block:: scala

  sealed trait Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] <: Up

     type Compare[N <: Nat] <: Comparison
  }

For _0#Compare, the resut is EQ if the other Nat is _0 and LT if it is not.

.. code-block:: scala

  sealed trait _0 extends Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] = IfZero

     type Compare[N <: Nat] =
        N#Match[ConstLT, EQ, Comparison]

     type ConstLT[A] = LT
  }

For Succ[N]#Compare, the result is GT if the other Nat is _0 and it recurses if it is not. The recursion compares O-1 to N. (Remember that N is Succ[N] – 1.)

.. code-block:: scala

  sealed trait Succ[N <: Nat] extends Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] = NonZero[N]

     type Compare[O <: Nat] =
        O#Match[N#Compare, GT, Comparison]
  }

This recursion is allowed because Compare implements an abstract type in a super-trait and the bound of that abstract type doesn’t change during recursion. We call N#Compare so we aren’t directly recursing, which wouldn’t be allowed.

Examples:

.. code-block:: scala

  scala> toBoolean[ _0#Compare[_0]#eq ]
  res0: true

  scala> toBoolean[ _0#Compare[_0]#lt ]
  res1: false

  scala> toBoolean[ _3#Compare[_4]#le ]
  res2: true

When we get to our binary search tree, Compare allows us to use Nats as keys in the tree. Next, we’ll define a general way to recurse on Nat using folds.
