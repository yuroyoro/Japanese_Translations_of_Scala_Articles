Type-Level Programming in Scala, Part 2: implicitly and =:=
-------------------------------------------------------------

Posted by Mark on June 10, 2010

This post briefly introduces a useful technique for comparing types (shown to me by Jason Zaugg) that will be used to check the results of type-level computations later.

It uses the ‘implicitly’ method, which is defined in Predef as:

.. code-block:: scala

  def implicitly[T](implicit e: T): T = e

This is useful for capturing an implicit value that is in scope and has type T.  The implicit that we want in this case is A =:= B for some types A and B.  A =:= B will only be found when A is the same type as B.

For example, this compiles:

.. code-block:: scala

  scala> implicitly[Int =:= Int]
  res0: =:=[Int,Int] = <function1>

but this does not:

.. code-block:: scala

  scala> implicitly[Int =:= String]
  error: could not find implicit value for parameter e: =:=[Int,String]

Also available are <:< and <%< for type conformance and views, respectively.

A conformance (<:<) example:

.. code-block:: scala

  scala> implicitly[Int =:= AnyVal]
  error: could not find implicit value for parameter e: =:=[Int,AnyVal]

  scala> implicitly[Int <:< AnyVal]
  res1: <:<[Int,AnyVal] = <function1>

A conversion (<%<) example:

.. code-block:: scala

  scala> implicitly[Int <:< Long]
  error: could not find implicit value for parameter e: <:<[Int,Long]

  scala> implicitly[Int <%< Long]
  res1: <%<[Int,Long] = <function1>

Note that when the compiler prints the <:< and <%< types, it doesn’t use infix notation. A small patch to the compiler can get us infix, however:

.. code-block:: scala

  scala> implicitly[Int <:< Long]
  error: could not find implicit value for parameter e: (Int <:< Long)

  scala> implicitly[Int <%< Long]
  res1: (Int <%< Long) = <function1>

It has a bit of a hack to print full names if the simple name is ambiguous. After 2.8.0.final is out, I’ll try to get feedback to improve it enough to be included in the standard compiler.
