Higher-Rank Polymorphism in Scala
-------------------------------------------------------------

Posted by Rúnar on July 2, 2010

I was reading Tim Carstens’s post about Haskell features he’d like to see in other languages, and one of those features was rank-2 types. Tim says that the killer application for this to ensure the safety of resource access, in the type system. In short, this feature enables us to make sure that if we have a value of a particular type, then it represents a system resource which can be safely accessed, and that it’s a compile-time type error to access a closed or unopened resource. I find it really fascinating that we can get this kind of assurance from a type system.

First, a brief explanation of rank-2 polymorphism. A regular (rank-1) polymorphic function in Scala might look like this:

.. code-block:: scala

  def apply[A](f: A => A, a: A): A = f(a)

This will apply the given function to a given value of any particular type, and it’s ensured that the argument and return types of the function are the same as the value to which it is applied. There’s only one type involved here, but it can be any particular type. But what if we wanted to say that the function f should work for all types? For example, a function that puts its argument in a list. Such a function should work for all types, as long as the input and output type match. For example:

.. code-block:: scala

  def singletonList[A](a: A): List[A] = List(a)

Now say we want to take such a function as an argument to another function. With just rank-1 polymorphism, we can’t do this:

.. code-block:: scala

  def apply[A,B](f: A => List[A], b: B, s: String): (List[B], List[String]) =
    (f(b), f(s))

It’s a type error because, B and String are not A. That is, the type A is fixed on the right of the quantifier [A,B]. We really want the polymorphism of the argument to be maintained so we can apply it polymorphically in the body of our function. Here’s how that might be expressed if Scala had rank-n types:

.. code-block:: scala

  def apply[B](f: (A => List[A]) forAll { A }, b: B, s: String): (List[B], List[String]) =
    (f(b), f(s))

That’s not legal Scala code, so let’s see how we could work around this. Note that a method can be polymorphic in its arguments, but a value of type Function1, Function2, etc, is monomorphic. So what we do is represent a rank-2 polymorphic function with a new trait that accepts a type argument in its apply method:

.. code-block:: scala

  trait ~>[F[_],G[_]] {
    def apply[A](a: F[A]): G[A]
  }

This trait captures something more general than Function1, namely a natural transformation from functor F to functor G. Note that A appears nowhere in the type and isn’t given until the apply method is actually called. We can now model a function that takes a value and puts it in a list, as a natural transformation from the identity functor to the List functor:

.. code-block:: scala

  type Id[A] = A

  val singletonList = new (Id ~> List) {
    def apply[A](a: A): List[A] = List(a)
  }

And we can now take such a function as an argument:

.. code-block:: scala

  def apply[B](f: Id ~> List, b: B, s: String): (List[B], List[String]) =
    (f(b), f(s))

I wonder if we can use this to get static guarantees of safe resource access, as in the SIO monad detailed in Lightweight Monadic Regions.
