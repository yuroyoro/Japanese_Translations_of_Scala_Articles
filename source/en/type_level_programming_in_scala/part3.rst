Type-Level Programming in Scala, Part 3: Boolean
-------------------------------------------------------------

Posted by Mark on June 13, 2010

A good introductory example to type-level programming is the Church encoding of booleans.  It doesn’t require anything too fancy and we need booleans when comparing numbers later.  If you are already familiar with the Church encoding of booleans, this post doesn’t add much, especially if you’ve seen them in Scala. This and the next post on type-level Peano numbers are intended for those unfamiliar with type-level programming.

See also: http://svn.assembla.com/svn/metascala/src/metascala/Booleans.scala.

The basic types involved are:

.. code-block:: scala

   sealed trait Bool
   sealed trait True extends Bool
   sealed trait False extends Bool

We can add conditional expressions by defining a type member If on Bool.  If accepts three parameters: the type to produce if the Bool is True, the type if it is False, and an upper bound that the first two types must conform to.  The upper bound is often important when working with the result of an If.

Our enhanced Bool looks like:

.. code-block:: scala

  sealed trait Bool {
   type If[T <: Up, F <: Up, Up] <: Up
  }

True and False simply return the appropriate argument:

.. code-block:: scala

  sealed trait True extends Bool {
   type If[T <: Up, F <: Up, Up] = T
  }
  sealed trait False extends Bool {
   type If[T <: Up, F <: Up, Up] = F
  }

Example usage:

.. code-block:: scala

  scala> type Rep[A <: Bool] = A#If[ Int, Long, AnyVal ]
  defined type alias Rep

  scala> implicitly[ Rep[True] =:= Int ]
  res1: =:=[Rep[Booleans.True],Int] = <function1>

  scala> implicitly[ Rep[False] =:= Int ]
  error: could not find implicit value for parameter e: =:=[Rep[Booleans.False],Int]

  scala> implicitly[ Rep[False] =:= Long ]
  res3: =:=[Rep[Booleans.False],Long] = <function1>

We can define some extra types in a Bool module:

.. code-block:: scala

  object Bool {
   type &&[A <: Bool, B <: Bool] = A#If[B, False, Bool]
   type || [A <: Bool, B <: Bool] = A#If[True, B, Bool]
   type Not[A <: Bool] = A#If[False, True, Bool]
  }

Example usage:

.. code-block:: scala

  scala> implicitly[ True && False || Not[False] =:= True ]

We can also add a method to the Bool module to convert a Bool type to a Boolean value for direct printing:

.. code-block:: scala

 def toBoolean[B <: Bool](implicit b: BoolRep[B]): Boolean = b.value

 class BoolRep[B <: Bool](val value: Boolean)
 implicit val falseRep: BoolRep[False] = new BoolRep(false)
 implicit val trueRep: BoolRep[True] = new BoolRep(true)

For example:

.. code-block:: scala

  scala> toBoolean[ True && False || Not[False] ]
  res0: Boolean = true

This is another method of checking the result of type-level computations.

Next up is type-level Peano numbers.
