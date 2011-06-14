Type-Level Programming in Scala, Part 7: Natural transformation literals
--------------------------------------------------------------------------------

Posted by Mark on October 26, 2010

The question we are trying to answer in this post is:
How easy can we make it to construct a natural transformation?

That is, we can define an ordinary anonymous function in Scala for rank-1 types using some succinct special syntax:

.. code-block:: scala

   val a: Char => Boolean = _.isUpper
   val b = (_: Char).isUpper
   val c = (c: Char) => c.isUpper

We run into problems when we proceed to natural transformations. We are not able to define a function that maps an Option[T] to List[T] for every T, for example. If this is not obvious, try to define 'toList' so that the following compiles:

.. code-block:: scala

   val toList = ...

   val a: List[Int] = toList( Some(3) )
   assert( List(3) == a )

   val b: List[Boolean] = toList( Some(true) )
   assert( List(true) == b )

In order to define a natural transformation M ~> N (here, M=Option, N=List), we have to create an anonymous class because Scala doesn’t have literals for quantified functions. So, we’ll explore in this post is how close we can get to a literal syntax.

First, one representation of a natural transformation in Scala looks like:

.. code-block:: scala

  trait ~>[-A[_], +B[_]] {
     def apply[T](a: A[T]): B[T]
  }

We use a strict argument here, but you could certainly make it call-by-name.

The filled in definition of toList in the motivating example using this representation is:

.. code-block:: scala

  val toList = new (Option ~> List) {
    def apply[T](opt: Option[T]): List[T] =
      opt.toList
  }

For any type T, this will turn an Option[T] into an List[T] by calling toList.

There is no simpler way to write this in plain Scala. We can’t write this as a function literal. There is no type that you can give a function literal such that you can apply 'toList' to an arbitrary 'Option[T]' and get back a 'List[T]'. For example:

.. code-block:: scala

  def toList[T](opt: Option[T]): List[T] = opt.toList
  val hOptFun = toList _

The type of hOptFun is actually 'Option[Nothing] => List[Nothing]' and not the quantified function '[T] Option[T] => List[T]'. Try applying hOptFun to a value:

.. code-block:: scala

  hOptFun(Some(3))
  error: type mismatch;
   found   : Int(3)
   required: Nothing
         hOptFun(Some(3))
                      ^

What can we do? Well, we can look to abstract types for some help. The idea is to define a trait representing a quantified type parameter where the quantified type is represented by an abstract type.

First, the trait:

.. code-block:: scala

  trait Param[A[_], B[_]] {
    type T
    def in: A[T]
    def ret(out: B[T]): Unit
    def ret: B[T]
  }

We will use this like so:

.. code-block:: scala

  val f = (p: Param[Option, List]) => p.ret( p.in.toList )

and define an implicit conversion from 'Param[A, B] => Unit' to 'A ~> B' :

.. code-block:: scala

  object Param {

    implicit def pToT[A[_], B[_]](p: Param[A,B] => Unit): A~>B = new (A ~> B) {
      def apply[s](a: A[s]): B[s] = {
        val v: Param[A,B] { type T = s} =

          new Param[A,B] { type T = s
            def in = a
            private var r: B[T] = _
            def ret(b: B[T]) {r = b}
            def ret: B[T] = r
          }

        p(v)
        v.ret
      }
    }
  }

We could add logic to the original scheme to ensure 'r' gets set exactly once in the mutable case, although this would be a runtime error and not a compile error. If we had dependent method and function types, we could keep it immutable:

.. code-block:: scala

  trait Param[A[_]] {
    type T
    def in: A[T]
  }
  object Param {
    implicit def pToT[A[_], B[_]](f: (p: Param[A]) => B[p.T]): A~>B = new (A ~> B) {
      def apply[s](a: A[s]): B[s] = {
        val v: Param[A] { type T = s} = new Param[A] { type T = s
          def in = a
        }
        f(v)
      }
    }
  }
   // usage:
  val f = (p: Param[Option, List]) => p.in.toList

So, we didn’t exactly succeed. In the absence of some help from the compiler, we’re stuck with some verbosity.

In part 8, we will apply natural transformations to higher-kinded heterogeneous lists. There it will be useful to define some auxiliary types and an identity transformation :

.. code-block:: scala

  object ~> {
     type Id[X] = X
     trait Const[A] { type Apply[B] = A }
     implicit def idEq : Id ~> Id = new (Id ~> Id) { def apply[T](a: T): T = a }
  }

Related links:
http://article.gmane.org/gmane.comp.lang.scala.user/697
http://existentialtype.net/2008/05/26/revisiting-higher-rank-impredicative-polymorphism-in-scala/
