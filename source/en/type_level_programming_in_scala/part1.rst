Part 1: Type Recursion
-------------------------------------------------------------

Scala’s type system in 2.8 allows recursion, and infinite recursion at that, as demonstrated in [1,2]. Arbitrary recursion is not allowed, as seen by ‘illegal cyclic reference’ errors. Perhaps allowing recursion is actually a bug, but we’ll try to do some interesting things while it is possible.

To use recursion, define a trait with a type declaration. A type declaration is an abstract type that can have type parameters and bounds. Then, define a subtrait that implements the type declaration. Recursion is allowed in this implementation, with restrictions that are discussed later.

Infinite Loop Example:

.. code-block:: scala

  // define the abstract types and bounds
  trait Recurse {
    type Next <: Recurse
    // this is the recursive function definition
    type X[R <: Recurse] <: Int
  }
  // implementation
  trait RecurseA extends Recurse {
    type Next = RecurseA
    // this is the implementation
    type X[R <: Recurse] = R#X[R#Next]
  }
  object Recurse {
    // infinite loop
    type C = RecurseA#X[RecurseA]
  }


Note that Int is arbitrary and could be replaced by any type. In general, this bound will be something meaningful. Like any other bound, it allows you to know something about the result without knowing the exact parameters provided.

There are some rules to follow to enable recursion. The upper bound can be a function of the type parameters, but it cannot change during recursion.

Consider:

.. code-block:: scala

  trait Recurse {
    type X[A, B] <: A
  }

for some types A and B. The implementation of X in some subtrait of Recurse cannot change A during recursion, but B can change. Concrete examples of this will be seen in later posts dealing with type-level numbers, HLists, and binary search trees.

Another restriction is that all recursion must be through a prefix, like a type projection:

.. code-block:: scala

  type X[R <: Recurse] = R#X[R#Next]

You cannot do direct recursion on a type member on the current type like this:

.. code-block:: scala

  type X[R <: Recurse] = X[R#Next]

So, with some reasonable restrictions, we have recursion and corecursion in the type-system, and thus full initial algebra semantics. Scala’s type system is a Turing-equivalent programming language.

- [1] http://apocalisp.wordpress.com/2009/09/02/more-scala-typehackery/
- [2] http://michid.wordpress.com/2010/01/29/scala-type-level-encoding-of-the-ski-calculus/
