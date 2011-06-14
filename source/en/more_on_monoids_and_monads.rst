More on Monoids and Monads
-------------------------------------------------------------

Posted by Rúnar on July 21, 2010

In a previous post on monoids, we established that Monoids are List-Algebras. In other words, a monoid A essentially models a function of type List[A] => A. We saw how the unit and associativity laws for monoids model the fact that we can fold left and right, and that the empty list doesn’t count.

Monoids for Free
_____________________________________________________________

We defined a monoid as a category with one object. Now, let’s generalize this notion. Before, we represented Monoid as having a binary function and an identity element for that function. But by our categorical definition, we could just restrict the Scala category (where types are the objects and functions are the arrows) and define Monoid in terms of that:

.. code-block:: scala

  def freeMonoid[M] = new Category[Function1] {
    def compose(m1: M => M, m2: M => M): M => M = m1 compose m2
    def identity: M => M = m => m
  }

A function from some type to itself is called an endofunction. The result of freeMonoid[M] for any type M, is a monoid, by our previous definition, because its only object is the set M and its only arrows are endofunctions on M.

We know already that lists and monoids are intimately connected. Now notice that M => M is exactly the form of the right-hand side of foldRight. Let’s remind ourselves of the type of foldRight:

.. code-block:: scala

  def foldRight[A,B]: (List[A], A => (B => B)) => (B => B)

Read this with an eye to endofunctions and the fact that they form monoids. FoldRight, then, is saying that a List[A] together with a function from a value of A to an endofunction on B, allows us to collapse all those As into a single endofunction on B. Here’s a possible implementation:

.. code-block:: scala

  def foldRight[A,B] = (as: List[A], f: A => B => B) => {
    val m = freeMonoid[B]
    as match {
      case Nil => m.identity
      case x :: xs => m.compose(f(x), foldRight(as, f))
    }

That’s slightly different from what you might be used to seeing, but equivalent.

A Theory of Monoids
_____________________________________________________________

This gives rise to a further theory. If we had a List[List[A]] such that A is a monoid, so we know we can go from List[A] to A, then it’s safe to say that we could fold all the inner lists and end up with List[A] which we can then fold. Another way of saying the same thing is that we can pass the list constructor :: to foldRight. It has the correct type for the f argument, which is not a coincidence (namely A => List[A] => List[A]). That in turn gives us another function (List[A] => List[A] => List[A]) which, as it turns out, appends one list to another. If we pass that to foldRight, we get List[List[A]] => List[A] => List[A]. That again has the proper type for foldRight, and so on. The pattern here is that we can keep doing this to turn an arbitrarily nested list of lists into an endofunction on lists, and we can always pass an empty list to one of those endofunctions to get an identity.

So the types of arbitrarily nested lists are monoids. Our composition for such a monoid appends one list to another, and the identity for that function is the empty list.

Let’s call this our “theory of monoids”, so we can refer to it by name later on.

Monoids of a Higher Kind
_____________________________________________________________

But there’s something more general going on here. For a moment, let’s denote List[A] as A*, meaning “zero or more values of type A“. If A is a monoid, then this gives us the function A* => A (since monoids are list-algebras). Now note what happened with arbitrary nesting of List. Let’s notate this as List*[A] meaning “lists of A nested to zero or more levels”. It seems that we can go from List*[A] => List[A]. So the List type constructor itself seems to be a sort of monoid of a higher kind.

And this turns out to be true. Let’s look at our monoid trait again:

.. code-block:: scala

  trait Monoid[M] {
    val identity: M
    def append(a: M, b: M): M
  }

With an eye towards M* => M, we can see that the Monoid provides a way of turning a list-of-M of arbitrary length (M*) into a list-of-M of length 1 (M¹). The identity takes care of the case where we have an empty list (M⁰), and the append reduces M² to M¹. By induction, we know that this is enough to collapse any number of Ms into a single M.

For our higher-kinded monoid, which we’ll call M as well, we need the same thing–a way of going from M⁰ to M¹, and from M² to M¹. “Higher-kinded monoid” is a bit long-winded. Since it’s a triad (a type constructor with identity and append), and also a sort of monoid, we’ll contract this to “Monad”:

.. code-block:: scala

  trait Monad[M[_]] {
    def map[A,B](f: A => B): M[A] => M[B]
    def unit[A](a: A): M[A]
    def join[A](m: M[M[A]]): M[A]
  }

The Obvious Axioms
_____________________________________________________________

Now recall the monoid laws from before. We can infer the monad laws from those. Monad’s unit is the identity of our higher-order monoid, and should be an identity for the join method, which is Monad’s equivalent of Monoid’s append. So join(unit(m)) == m (left identity) and join(map(unit)(m)) == m (right identity).

The join method should be associative, meaning that if you have M[M[M[A]]], it doesn’t matter whether you join the inner or outer Ms first. That is, join(map(join)(m)) should equal join(join(m)).

This is pretty much a straight translation from the monoid laws, except that we needed map to apply a function “on the right” (or on the inside) of an M.

The Monad for Monoids
_____________________________________________________________

OK, so remember our theory of monoids from above? We can encode that theory in Scala, using a monad:

.. code-block:: scala

  val listMonad = new Monad[List] {
    def map[A,B](f: A => B) = (xs: List[A]) => xs.map(f)
    def unit[A](a: A) = a :: Nil
    def join[A](xs: List[List[A]]) = xs.flatten
  }

So List is a monad that represents a theory about monoids. What is this thing saying about monoids? The map method says that if we can fold a List[A] with a monoid, and we can turn every A into a monoid B, then we can fold a List[B]. The unit and join methods, together with the monad laws above, are saying the following:

If we have an expression of some monoid-type A, like (a1 + a2 + a3), where a1, a2, and a3 are values of type A and + is the append function of the monoid, then we can insert the identity into that expression wherever we want, and we can add and remove parentheses at will, and the order in which we add and remove them doesn’t matter.

Monads Are Not Just About Monoids
_____________________________________________________________

Alright, so the List monad is all about monoids, but this is just one use case for the Monad trait. I.e. List is the monad for monoids.

We could write a Monad instance that means something totally different. For example we might write Monad[Future], to encode a theory of concurrent processes. What would that theory be saying? Well, map would say that we can extend a running process by giving its continuation. And the unit and join functions with the monad laws would say that we can arbitrarily fork subexpressions of a program, and arbitrarily join with forked processes, and that the order in which we fork and join doesn’t matter to the result of the program.

We can perfectly well write an implementation of Monad[Option], or Monad[Reader[A]#->] where trait Reader[A] { type ->[B] = A => B }. I’ll leave these implementations as an exercise for the reader. See if you can come up with an explanation for the monad laws in each case.

I hope you’ve enjoyed this deep dive into monoids, monads, and their relationship.
