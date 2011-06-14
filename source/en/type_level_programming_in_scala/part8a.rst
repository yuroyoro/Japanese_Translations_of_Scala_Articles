Type-Level Programming in Scala, Part 8a: KList motivation
-------------------------------------------------------------

Posted by Mark on November 1, 2010

In part 8, we will look at operations on arbitrary arity tuples over a type constructor. These are higher-kinded heterogeneous lists, which I’ll KLists. To motivate why we might want such a structure, we’ll start with Tuple2 for simplicity.

Consider some basic transformations on the elements of a Tuple2. If we want to apply the same function to both elements, we can require that the elements have a common supertype and that the function operates on that supertype:

.. code-block:: scala

  def map1[A <: C,B <: C,C,D](t: (A,B), f: C => D): (D,D) =
   (f(t._1), f(t._2))

  scala> map1( ("3", true), (_: Any).hashCode )
  res1: (Int, Int) = (51,1231)

Or, we might want to supply two separate functions to operate on each component independently:

.. code-block:: scala

  def map2[A,B,C,D](t: (A,B), f: A => C, g: B => D): (C,D) =
    (f(t._1), g(t._2))

  scala> map2( (1, true), (_: Int) + 1, ! (_: Boolean) )
  res3: (Int, Boolean) = (2,false)

Now, consider a Tuple2 where the types of both components are created by the same type constructor, such as (List[Int], List[Boolean]) or (Option[String], Option[Double]).

One useful operation on such a structure looks like:

.. code-block:: scala

  def transform[F[_], A, B, G[_]] (k: (F[A], F[B]), g: F ~> G): (G[A], G[B]) =
    ( g(k._1), g(k._2) )

This applies the provided natural transformation to each element, preserving the underlying type parameter, but with a new type constructor. As an example, we can apply the toList transformation we defined in part 7:

.. code-block:: scala

  val toList = new (Option ~> List) {
    def apply[T](opt: Option[T]): List[T] =
      opt.toList
  }

  // these are so that we get Option[T] as the type
  // and not Some[T] or None
  def some[T](t: T): Option[T] = Some(t)
  def none[T]: Option[T] = None

  scala> transform((some(3), some("str")), toList)
  res5: (List[Int], List[java.lang.String]) = (List(3),List(str))

  scala> transform((some(true), none[Double]), toList)
  res6: (List[Boolean], List[Double]) = (List(true),List())

In part 6, we were concerned with the generalization of TupleN to arbitrary arity. In part 8, we will generalize ‘transform’ and related operations to heterogeneous lists over a type constructor.
