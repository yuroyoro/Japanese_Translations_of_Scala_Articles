Type-Level Programming in Scala, Part 5a: Binary numbers
-------------------------------------------------------------

Posted by Mark on June 24, 2010

The Nat operations are generally limited to results under 10,000. Compilation time gets quite long or the stack overflows beyond that. An alternative is to use a binary representation of numbers, called Dense here after the implementation in Okasaki’s Purely Functional Data Structures. We will use this representation to solve a simplified Project Euler problem in the type system in the next post.

Only the basic form and the implementations of increment and shift are shown here. See the full source for Add, Mult, and Exp.

First, we need a Digit type to represent a bit. It has two subtypes, One and Zero. We define a type member to Match on the subtype and a Compare member for comparing digits.

.. code-block:: scala

  sealed trait Digit {
     type Match[ IfOne <: Up, IfZero <: Up, Up] <: Up
     type Compare[D <: Digit] <: Comparison
  }
  sealed trait Zero extends Digit {
     type Match[ IfOne <: Up, IfZero <: Up, Up] = IfZero
     type Compare[D <: Digit] = D#Match[ LT, EQ, Comparison]
  }
  sealed trait One extends Digit {
     type Match[ IfOne <: Up, IfZero <: Up, Up] = IfOne
     type Compare[D <: Digit] = D#Match[ EQ, GT, Comparison]
  }

For example:

.. code-block:: scala

  type Is0[D <: Digit] = D#Match[ False, True, Bool ]
  implicitly[ Is0[Zero] =:= True ]
  implicitly[ One#Compare[Zero] =:= GT ]

Next, we create the Dense type. It is a type level heterogeneous list with element types constrained to be Digits. The head of the list is the least significant bit. The last element of a non-empty list is always One and is the most significant bit. An empty list represents zero.

.. code-block:: scala

  sealed trait Dense {
     type digit <: Digit
     type tail <: Dense
     type Inc <: Dense
     type ShiftR <: Dense
     type ShiftL <: Dense
  }
  sealed trait DCons[d <: Digit, T <: Dense] extends Dense {
     type digit = d
     type tail = T
     type Inc = d#Match[ Zero :: T#Inc, One :: T, Dense ]
     type ShiftR = tail
     type ShiftL = Zero :: DCons[d, T]
  }
  sealed trait DNil extends Dense {
     type tail = Nothing
     type digit = Nothing
     type Inc = One :: DNil
     type ShiftR = DNil
     type ShiftL = DNil
  }

We see that shifts come easily with Dense (compared to Nat). It is just the tail of the list (shift right) or prepending a zero (shift left).

We can predefine some integers (using the alias :: for DCons):

.. code-block:: scala

   type ::[H <: Digit, T <: Dense] = DCons[H, T]

   type _0 = DNil
   type _1 = One :: DNil
   type _2 = Zero :: One :: DNil
   type _3 = One :: One :: DNil
   type _4 = Zero :: Zero :: One :: DNil
   type _5 = One :: Zero :: One :: DNil
   ...
As usual, we define a conversion to values:

.. code-block:: scala

   def toInt[ D <: Dense](implicit drep: DRep[D]): Int = drep.value

   final class DRep[D <: Dense](val value: Int)

   implicit def dnilToRep = new DRep[DNil](0)
   implicit def dcons0ToRep[D <: Dense](implicit tailRep: DRep[D]): DRep[DCons[Zero, D]] = new DRep(tailRep.value * 2)
   implicit def dcons1ToRep[D <: Dense](implicit tailRep: DRep[D]): DRep[DCons[One, D]] = new DRep(tailRep.value * 2 + 1)

See Dense.scala for the implementation of other operations, like addition and multiplication. Example usage:

.. code-block:: scala

  toInt[ _14 ] == 14
  toInt[ _5#Inc ] == 6
  toInt[ _5#Add[_7] ] == 12
  toInt[ _12#Mult[_8]#Mult[_13] ] == 1248
  toInt[ _4#Exp[_15] ] == 1073741824

The last line compiles in under a second, which shows that higher numbers are handled better using this representation and converting them to actual integer values is not a problem. (Ok, it helped that 4 is a power of two. _3#Exp[_8] is actually more problematic for Dense than Nat.)

Dense will make a good example for keys in a type-level map later. Next, we’ll solve a (much) smaller version of a Project Euler problem using these type-level binary numbers. We will need Match and Compare defined on Dense for that (implementations on github):

.. code-block:: scala

  type Match[ NonZero <: Up, Zero <: Up, Up] <: Up
  type Compare[B <: Dense] <: Comparison
