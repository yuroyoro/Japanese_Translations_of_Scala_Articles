Type-Level Programming in Scala, Part 8c: KList ZipWith
-------------------------------------------------------------

Posted by Mark on November 15, 2010

One use of KList and the ‘transform’ and ‘down’ methods from 8b is to implement methods like ‘zipWith’ for arbitrary tuple lengths. To start with, the signature of zipWith for a pair of Streams, operating on a fixed arity of 2, looks like:

.. code-block:: scala

  def zipWith2[A,B,C](t2: (Stream[A], Stream[B]))(f: (A,B) => C): Stream[C] =
     t2 match
     {
        case (ha #:: ta, hb #:: tb) => Stream.cons( f(ha, hb), zipWith2( (ta, tb) )(f) )
        case _ => Stream.empty
     }

Example usage:

.. code-block:: scala

  val nats = Stream.from(1)
  val random = Stream.continually( math.random )
  val seq = zipWith2( (nats, random) ) { (n, r) => if(r > 0.3) n else -n }

  scala> seq.take(10).toList
  res0: List[Int] = List(-1, 2, 3, 4, 5, 6, -7, -8, 9, 10)

For the implementation of zipWith2, if either Stream in the pair is empty, the resulting Stream is empty. Otherwise, there is a head element for each stream in the pair. We apply the provided function to these elements and make the result the head of a new Stream. The tail of this new Stream will be the result of recursing on the tails of the input pair.

To generalize this to arbitrary arity, we will operate on a KList of Streams. Because we want to abstract over arity, we use a heterogeneous list. We use KList instead of HList because we want to constrain each element in the tuple to be a Stream and we don’t care what the specific types of Streams the elements are, but we do want to preserve those types. When we take the head element of each Stream, the resulting list is the underlying HList type of the input KList. For example, given an input of type KList[Stream, A :: B :: HNil], when we take the head of each Stream in the KList we will get an HList of type A :: B :: HNil. This is like going from (Stream[A], Stream[B]) to (A,B).

So, if we end up with the underlying HList type, the function we will apply to the input KList must be a function from that HList type to some other type. In the example above, the function type would be A :: B :: HNil => T for some type T, which will be the type of the output Stream. With this, we have our signature for a generalized zipWith:

.. code-block:: scala

  def zipWith[HL <: HList, T](kl: KList[Stream, HL])(f: HL => T): Stream[T]

To implement this function, we again break the problem into two parts. If any Stream is empty, the resulting stream is empty. Otherwise, we get all the head elements of the Streams as an HList, apply the input function to it, and make this the new head. For the tail, we get all of the tails of the Streams and recurse. To get the head elements, we use ‘down’ because we want KList[Stream, HL] => HL. For the tails, we use 'transform' because we need a mapping KList[Stream, HL] => KList[Stream, HL]. The implementation looks like:

.. code-block:: scala

  def zipWith[HL <: HList, T](kl: KList[Stream, HL])(f: HL => T): Stream[T] =
     if(anyEmpty(kl))
        Stream.empty
     else
        Stream.cons( f( kl down heads ), zipWith(kl transform tails )( f ) )

  def anyEmpty(kl: KList[Stream, _]): Boolean = kl.toList.exists(_.isEmpty)

  val heads = new (Stream ~> Id) { def apply[T](s: Stream[T]): T = s.head }
  val tails = new (Stream ~> Stream) { def apply[T](s: Stream[T]): Stream[T] = s.tail }

The toList function on KList has type KList[M, HL] => List[M[_]] and has a trivial implementation. Since List is homogeneous, we can’t preserve each individual cell’s type, but we can at least use the common type constructor. In ‘zipWith’, this means we can call the ‘isEmpty’ method on the elements of the list but we would not get a very specific type if we called ‘head’, for example. ‘heads’ and ‘tails’ are natural transformations that map a Stream[T] to its head of type T and its tail of type Stream[T], respectively.

The original example translated to use the generalized zipWith looks like:

.. code-block:: scala

  val nats = Stream.from(1)
  val random = Stream.continually( math.random )
  val seq = zipWith( nats :^: random :^: KNil ) {
     case n :: r :: HNil => if(r > 0.3) n else -n
  }

  scala> seq.take(10).toList
  res0: List[Int] = List(-1, 2, -3, -4, -5, 6, 7, 8, 9, 10)

We can implement the related ‘zipped’ function in terms of ‘zipWith’.

.. code-block:: scala

  def zipped[HL <: HList](kl: KList[Stream, HL]): Stream[HL] =
     zipWith(kl)(x => x)

Or, we could have implemented zipWith in terms of zipped. In any case, we can implement several other functions using zipped:

.. code-block:: scala

  def foreach[HL <: HList, T](kl: KList[Stream, HL])(f: HL => T): Unit =
     zipped(kl).foreach(f)

  def collect[HL <: HList, T](kl: KList[Stream, HL])(f: HL => Option[T]): Stream[T] =
     zipped(kl).collect(f)

  def flatMap[HL <: HList, T](kl: KList[Stream, HL])(f: HL => Stream[T]): Stream[T] =
     zipped(kl).flatMap(f)

  def forall[HL <: HList](kl: KList[Stream, HL])(f: HL => Boolean): Boolean =
     zippped(kl).forall(f)

  def exists[HL <: HList](kl: KList[Stream, HL])(f: HL => Boolean): Boolean =
     zipped(kl).exists(f)

An example using ‘foreach’:

.. code-block:: scala

  val a = Stream(1,2,5,3,9,10,101)
  val b = Stream("one", "two", "three", "four")
  val c = Stream(true, false, false, true, true)

  zipped(a :^: b :^: c :^: KNil) foreach {
     case n :: s :: b :: HNil =>
        println( s * (if(b) 1 else n) )
  }

  one
  twotwo
  threethreethreethreethree
  four
