On Monoids
-------------------------------------------------------------

Posted by Rúnar on June 14, 2010

I’m often asked about monoids, monads, and what the difference is between the two. Seeing that there’s not a lot of material that explains these things in terms of Scala, I’m going to try to remedy that presently. Even if you already know what these things are, I hope to provide a little deeper insight. In this post I’m going to talk about monoids in some depth.

First, here is a definition for Monoid, in Scala:

.. code-block:: scala

  trait Monoid[T] {
    def append(m1: T, m2: T): T
    val identity: T
  }

To state that in English, a monoid is given by a type T together with a function append: (T, T) => T, and a value identity: T. The rules for a monoid are that append must be associative (meaning that append(x, append(y, z)) == append(append(x, y), z)), and identity must be an identity for that function (such that append(identity, x) == x, and append(x, identity) == x. For example, String with concatenation and the empty string form a monoid. Boolean with true and &&, or Boolean with false and ||, also are monoids. As are Int with addition and 0, multiplication and 1, etc.

As an example, here is the String monoid:

.. code-block:: scala

  val stringMonoid = new Monoid[String] {
    def append(s1: String, s2: String) = s1 + s2
    val identity = ""
  }

Monoids are List-Algebras
_____________________________________________________________

Everyone is familiar with the singly-linked list (a.k.a. cons-list). A List[T] is constructed as either the empty list (Nil), or from an element of type T and another List[T].

Nil is an irreducible unit (it takes no arguments and has no structure), and is in fact totally equivalent to the () value of type Unit. The :: (cons) constructor takes two arguments, and indeed a non-empty List[T] is equivalent to a pair (T, List[T]) (the head and the tail).

Note that a Monoid consists of two functions, one that takes no arguments, and one that takes two arguments, just like the constructors of List. This is not a coincidence. If you have a List[T] and a Monoid[T], you can reduce the list using the Monoid, to a single value of type T.

.. code-block:: scala

  def reduce[A](ts: List[A], m: Monoid[A]): A =
    ts match {
      case Nil => m.identity
      case x :: xs => m.append(x, reduce(xs))
    }

If you expand out the Monoid argument, you get:

def reduce[A](ts: List[A], identity: A, append: (A, A) => A): A
This looks suspicously similar to foldRight and foldLeft. Again, not a coincidence. Here’s the signature for foldRight:

def foldRight[A,B](ts: List[A], identity: B, append: (A, B) => B): B
In fact, the implementation of these last two is exactly identical. So, monoids are just an abstraction that represents the arguments you give to folds. The associativity rule simply models the fact that you should get the same answer for a given monoid whether you foldLeft or foldRight. The identity rule (in loose terms) models the fact that the empty list “doesn’t count”.

Monoids are one-object categories
_____________________________________________________________

Those readers further along might be wondering where this notion of monoids comes from, and why the concept is even warranted in the first place. Well, it comes from Category Theory. So let me give you a crash course.

A category consists of objects (A, B, C, etc.) and arrows between objects, (A => B, A => C, B => C, etc). You can think of it as a diagram or graph, with objects as the nodes, and arrows as directed edges between the nodes. Given any pair of arrows f: B => C and g: A => B, there exists a composite arrow (f * g): A => C.

Here’s an encoding of the concept in Scala:

.. code-block:: scala

  trait Category[F[_,_]] {
    def compose[A,B,C](f: F[B,C], g: F[A,B]): F[A,C]
    def identity[A]: F[A,A]
  }

For example, there’s a category where the objects are Scala types, the arrows are Scala functions, and the composition of arrows is simply function composition.

.. code-block:: scala

  val catScala = new Category[Function1] {
    def compose[A,B,C](f: B => C, g: A => B): A => C =
      f compose g
    def identity[A]: A => A =
      (a: A) => a
  }

A category must have an identity arrow for each object, which points from the object to itself. In the Scala category, these arrows are represented by the identity function.

Furthermore, a category must satisfy associativity ((f * g) * h == f * (g * h)), and identity (f * identity == f and identity * f == f), for all arrows f, g, and h.

Note that the laws that a category must satisfy are exactly the laws that a monoid must satisfy. You can think of the append function of the monoid as the arrow composition, and the identity value as the identity arrow. For a Monoid[T], the arrows are precisely all the values of type T, and T itself is the only object in the category.

We can give this category in Scala:

.. code-block:: scala

  // The category for a given monoid.
  def monoidCategory[M](m: Monoid[M]) =
    new Category[({type λ[α, β] = M})#λ] {
      def identity[A] = m.identity
      def compose[X, Y, Z](f: M, g: M) = m.append(f, g)
    }
