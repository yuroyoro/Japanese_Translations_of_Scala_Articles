Towards an Effect System in Scala, Part 1: ST Monad
-------------------------------------------------------------

Posted by Rúnar on March 20, 2011

Referentially Transparent Mutable State
_____________________________________________________________

In their paper “Lazy Functional State Threads”, John Launchbury and Simon Peyton-Jones present a way of securely encapsulating stateful computations that manipulate mutable objects. The result is Haskell’s ST monad. Its definition is very similar to the State data type. In Haskell, the ST monad is used to thread the manipulation of mutable state in such a way that the mutation is completely referentially transparent, because it is a type error for a mutable object to escape the monad.

I would like to present an implementation of this in Scala, which I recently committed to the Scalaz library. I was inspired to write this by Tim Carstens last summer, but never found a way of encoding the requisite rank-2 types in Scala’s type system in such a way that what should work does and what shouldn’t doesn’t. But Geoff Washburn got me going again. Following the technique on his blog, of representing universal quantifiers as doubly negated existentials, I was able to encode ST in a way that’s surprisingly nice to use, and actually does give you type errors if you try to access a naked mutable reference. And as Mark Harrah has pointed out, we end up not having to use the double negation after all. I’m surprised to find that doing this in the obvious way in Scala, just works.

OK, let’s get to the money. In Scala, we can declare the ST data type as follows:

.. code-block:: scala

  case class World[A]()

  case class ST[S, A](f: World[S] => (World[S], A)) {
    def apply(s: World[S]) = f(s)
    def flatMap[B](g: A => ST[S, B]): ST[S, B] =
      ST(s => f(s) match { case (ns, a) => g(a)(ns) })
    def map[B](g: A => B): ST[S, B] =
      ST(s => f(s) match { case (ns, a) => (ns, g(a)) })
  }

  def returnST[S, A](a: => A): ST[S, A] = ST(s => (s, a))

This is a monad in the obvious way. The flatMap method is monadic bind and returnST is monadic unit.

The World type represents some state of the world, and the ST type encapsulates a state transformer which receives the state of the world and returns a value which depends on that state together with a new state. Here, we are representing the world state by nothing at all. It turns out that for what we want to do with the ST monad, the contents of the state are not important, but its type very much is. A much more detailed explanation of how and why this works is given in the paper, but the punchline is that we are going to “transform the state” by mutating objects in place, and in spite of this the state transformer is going to be a pure function. This is achieved by guaranteeing that the type S for a given state transformer is unique. More on that in a second.

Purely Functional Mutable References
_____________________________________________________________

A simple object that we can mutate in place is one that holds a reference to another object through a mutable variable.

.. code-block:: scala

  case class STRef[S, A](a: A) {
    private var value: A = a

    def read: ST[S, A] = returnST(value)
    def write(a: A): ST[S, STRef[S, A]] = ST((s: World[S]) => {value = a; (s, this)})
    def mod[B](f: A => A): ST[S, STRef[S, A]] = for {
      a <- read
      v <- write(f(a))
    } yield v
  }

  def newVar(a: => A) = returnST(STRef(a))

So we have monadic combinators to construct, read, write, and modify references. Note that the implementation of write blatantly mutates the object in place. The definition of mod shows how to compose state transformers in sequence, using monad comprehensions.

It’s important that an STRef is parameterized on a type S which represents the state thread that created it. This makes variables allocated by different state threads have incompatible types. Therefore, state threads cannot ever see each other’s mutable variables. Because state transformers can only be composed sequentially (with flatMap), it’s guaranteed that two of them can never simultaneously mutate the same STRef.

Running a State Transformer as a Pure Function
_____________________________________________________________

Note that the type of a reference to a value of type A in a state thread S is ST[S, STRef[S, A]]. If ST had a run function of type ST[S, A] => A, we would be able to get the reference out. But this type is more general than we want. What we want is for the compiler to reject code like newVar(10).run, which would give you access to the naked STRef, but to accept code like newVar(10).flatMap(_.mod(x => x + 1).flatMap(read)).run, which simply accesses an integer.

In Haskell, the type of runST is:

.. code-block:: haskell

  runST :: forall a. (forall s. ST s a) -> a.

This is a rank-2 type which Scala’s type system does not directly support.

To see why this type would prevent the leaking of a mutable reference, consider the type you would need in order to get an STRef out of the ST monad.

.. code-block:: haskell

  forall a. (forall s. ST s (STRef s a)) -> STRef ??? a

What type should go in place of the three question marks? There is no type that could possibly fit the bill because the type s is bound (introduced) by the universal quantifier to the left of the arrow. It’s a local type variable in the domain of the function, so it can’t escape to the codomain. This is why ST state transformers are referentially transparent.

Of course, if you get the value out of a reference, then you can run that just fine. In Scala terms, you can always go from ST[S, A] to A, but you can never go from ST[S, F[S]] to F[S] for any F[_].

Writing runST in Scala
_____________________________________________________________

So the problem becomes how to represent a rank-2 polymorphic type in Scala. I’ve shown before how we can represent a rank-2 function type by encoding it as a natural transformation. And Mark has posted on how to write natural transformations using universally quantified values. (And I just now realized that he’s using functional state threads for non-observable mutation!)

First, we need a representation of universally quantified values:

.. code-block:: scala

  trait Forall[P[_]] {
    def apply[A]: P[A]
  }

Now that we have rank-2 polymorphism, the implementation of runST is straightforward:

.. code-block:: scala

    def runST[A](f: Forall[({type λ[S] = ST[S, A]})#λ]): A =
      f.apply.f(realWorld)._2

I’m using the “type lambda” trick here to declare the type constructor inline. The realWorld object is just a dummy value.

Some Examples
_____________________________________________________________

Here’s a simple example of a computation that creates a mutable reference and mutates it:

.. code-block:: scala

  def e1[S]: ST[S, STRef[S, Int]] = for {
    r <- newVar[S, Int](0)
    x <- r.mod(_ + 1)
  } yield x

And this expression creates a reference, mutates it, and then reads the value out:

.. code-block:: scala

  def e2[A] = e1[A].flatMap(_.read)

Running the latter expression is fine, since it just returns an Int:

.. code-block:: scala

  runST(new Forall[A] { def apply[A] = e2 })

But running the former fails at compile-time because it exposes a mutable reference. Or rather, because when the compiler tries to unify with our existential type, it’s out of scope:

.. code-block:: scala

  runST(new Forall[({type λ[S] = ST[S, STRef[S, Int]]})#λ] { def apply[A] = e1 })

  found   : scalaz.Forall[[S(in type λ)]scalaz.ST[S(in type λ),scalaz.STRef[S(in type λ),Int]]]
  required: scalaz.Forall[[S(in type λ)]scalaz.ST[S(in type λ),scalaz.STRef[_ >: (some other)S(in type λ) with (some other)S(in type λ), Int]]]

What are the practical implications of this kind of compile-time checking? I will just quote Peyton-Jones and Launchbury:

It is possible to encapsulate stateful computations so that they appear to the rest of the program as pure (stateless) functions which are guaranteed by the type system to have no interactions whatever with other computations, whether stateful or otherwise (except via the values of arguments and results, of course).

Complete safety is maintained by this encapsulation. A program may contain an arbitrary number of stateful sub-computations, each simultaneously active, without concern that a mutable object from one might be mutated by another.

This can be taken much further than these simple examples. In Scalaz, we have STArrays, which are purely functional mutable arrays. There’s an example of a pure binsort which uses a mutable array for sorting.

This technique can be extrapolated to implement Monadic Regions (currently underway for Scalaz), which allows compile-time tracking of not just mutable arrays and references, but file handles, database connections, and any other resource we care to track.

What we have here then is essentially the beginnings of an effect system for Scala. This allows us to compose programs from referentially transparent components which are internally implemented with mutation and effects, while those effects are guaranteed by the type system to be transparent to the rest of the program.
