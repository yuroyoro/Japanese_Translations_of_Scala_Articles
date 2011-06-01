Scalaz Tutorial: Enumeration-Based I/O with Iteratees
=============================================================
Posted by Rúnar on October 17, 2010

Scalaz 5.0 adds an implementation of a concept called Iteratee. This is a highly flexible programming technique for writing enumeration-based input processors that can be freely composed.

A lot of people have asked me to write a tutorial on how this works, specifically on how it is implemented in Scalaz and how to be productive with it, so here we go.

The implementation in Scalaz is based on an excellent article by John W. Lato called “Iteratee: Teaching an Old Fold New Tricks”. As a consequence, this post is also based on that article, and because I am too unoriginal to come up with my own examples, the examples are directly translated from it. The article gives code examples in Haskell, but we will use Scala here throughout.

Motivation
-------------------------------------------------------------

Posted by Rúnar on October 17, 2010

Most programmers have come across the problem of treating an I/O data source (such as a file or a socket) as a data structure. This is a common thing to want to do. To contrast, the usual means of reading, say, a file, is to open it, get a cursor into the file (such as a FileReader or an InputStream), and read the contents of the file as it is being processed. You must of course handle IO exceptions and remember to close the file when you are done. The problem with this approach is that it is not modular. Functions written in this way are performing one-off side-effects. And as we know, side-effects do not compose.

Treating the stream of inputs as an enumeration is therefore desirable. It at least holds the lure of modularity, since we would be able to treat a File, from which we’re reading values, in the same way that we would treat an ordinary List of values, for example.

A naive approach to this is to use iterators, or rather, Iterables. This is akin to the way that you would typically read a file in something like Ruby or Python. Basically you treat it as a collection of Strings:

.. code-block:: scala

  def getContents(fileName: String): Iterable[String] = {
    val fr = new BufferedReader(new FileReader(fileName))
    new Iterable[String] {
      def iterator = new Iterator[String] {
        def hasNext = line != null
        def next = {
          val retVal = line
          line = getLine
          retVal
        }
        def getLine = {
          var line:String = null
          try {
            line = fr.readLine
          } catch {
            case _ => line = null
          }
          line
        }
        var line = getLine
      }
    }
  }


What this is doing is a kind of lazy I/O. Nothing is read from the file until it is requested, and we only hold one line in memory at a time. But there are some serious issues with this approach. It’s not clear when you should close the file handle, or whose responsibility that is. You could have the Iterator close the file when it has read the last line, but what if you only want to read part of the file? Clearly this approach is not sufficient. There are some things we can do to make this more sophisticated, but only at the expense of breaking the illusion that the file really is a collection of Strings.

The Idea
-------------------------------------------------------------

Any red-blooded functional programmer should be thinking right about now: “Instead of getting Strings out of the file, just pass in a function that will serve as a handler for each new line!” Bingo. This is in fact the plot with Iteratees. Instead of implementing an interface from which we request Strings by pulling, we’re going to give an implementation of an interface that can receive Strings by pushing.

And indeed, this idea is nothing new. This is exactly what we do when we fold a list:

.. code-block:: scala

  def foldLeft[B](b: B)(f: (B, A) => B): B


The second argument is exactly that, a handler for each element in the list, along with a means of combining it with the accumulated value so far.

Now, there are two issues with an ordinary fold that prevent it from being useful when enumerating file contents. Firstly, there is no way of indicating that the fold should stop early. Secondly, a list is held all in memory at the same time.

The Iteratee Solution
-------------------------------------------------------------

Scalaz defines the following two data structures (actual implementation may differ, but this serves for illustration):

.. code-block:: scala

  trait Input[+E]
  case class El[E](e: E) extends Input[E]
  case object Empty extends Input[Nothing]
  case object EOF extends Input[Nothing]

  trait IterV[E,A] {
    def run: A = ... // Implementation omitted
  }
  case class Done[E,A](a: A, e: Input[E]) extends IterV[E,A]
  case class Cont[E,A](k: Input[E] => IterV[E,A]) extends IterV[E,A]


So an input to an iteratee is represented by Input[E], where E is the element type of the input source. It can be either an element (the next element in the file or stream), or it’s one of two signals: Empty or EOF. The Empty signal tells the iteratee that there is not an element available, but to expect more elements later. The EOF signal tells the iteratee that there are no more elements to be had.

Note that this particular set of signals is kind of arbitrary. It just facilitates a particular set of use cases. There’s no reason you couldn’t have other signals for other use cases. For example, a signal I can think of off the top of my head would be Restart, which would tell the iteratee to start its result from scratch at the current position in the input.

IterV[E,A] represents a computation that can be in one of two states. It can be Done, in which case it will hold a result (the accumulated value) of type A. Or it can be waiting for more input of type E, in which case it will hold a continuation that accepts the next input.

Let’s see how we would use this to process a List. The following function takes a list and an iteratee and feeds the list’s elements to the iteratee.


.. code-block:: scala

  def enumerate[E,A]: (List[E], IterV[E,A]) => IterV[E,A] = {
    case (Nil, i) => i
    case (_, i@Done(_, _)) => i
    case (x :: xs, Cont(k)) => enumerate(xs, k(El(x)))
  }


Now let’s see some actual iteratees. As a simple example, here is an iteratee that counts the number of elements it has seen:

.. code-block:: scala

  def counter[A]: IterV[A,Int] = {
    def step(n: Int): Input[A] => IterV[A,Int] = {
      case El(x) => Cont(step(n + 1))
      case Empty => Cont(step(n))
      case EOF => Done(n, EOF)
    }
    Cont(step(0))
  }

And here’s an iteratee that discards the first n elements:

.. code-block:: scala

  def drop[E,A](n: Int): IterV[E,Unit] = {
    def step: Input[E] => IterV[E,Unit] = {
      case El(x) => drop(n - 1)
      case Empty => Cont(step)
      case EOF => Done((), EOF)
    }
    if (n == 0) Done((), Empty) else Cont(step)
  }

And one that takes the first element from the input:

.. code-block:: scala

  def head[E]: IterV[E, Option[E]] = {
    def step: Input[E] => IterV[E, Option[E]] = {
      case El(x) => Done(Some(x), Empty)
      case Empty => Cont(step)
      case EOF => Done(None, EOF)
    }
    Cont(step)
  }


Let’s go through this code. Each one defines a “step” function, which is the function that will handle the next input. Each one starts the iteratee in the Cont state, and the step function always returns a new iteratee in the next state based on the input received. Note in the last one (head), we are using the Empty signal to indicate that we want to remove the element from the input. The utility of this will be clear when we start composing iteratees.

Now, an example usage. To get the length of a list, we write:

.. code-block:: scala

  val length: Int = enumerate(List(1,2,3), counter[Int]).run // 3


The run method on IterV just gets the accumulated value out of the Done iteratee. If it isn’t done, it sends the EOF signal to itself first and then gets the value.

Composing Iteratees
-------------------------------------------------------------

Notice a couple of things here. With iteratees, the input source can send the signal that it has finished producing values. And on the other side, the iteratee itself can signal to the input source that it has finished consuming values. So on one hand, we can leave an iteratee “running” by not sending it the EOF signal, so we can compose two input sources and feed them into the same iteratee. On the other hand, an iteratee can signal that it’s done, at which point we can start sending any remaining elements to another iteratee. In other words, iteratees compose sequentially.

In fact, IterV[E,A] is an instance of the Monad type class for each fixed E, and composition is very similar to the way monadic parsers compose:

.. code-block:: scala

  def flatMap[B](f: A => IterV[E,B]) = this match {
    case Done(x, e) => f(x) match {
      case Done(y, _) => Done(y, e)
      case Cont(k) => k(e)
    }
    case Cont(k) => Cont(e => k(e) flatMap f)
  }

Here then is an example of composing iteratees with a for-comprehension:

.. code-block:: scala

  def drop1Keep1[E]: IterV[E, Option[E]] = for {
    _ <- drop(1)
    x <- head
  } yield x


The iteratee above discards the first element it sees and returns the second one. The iteratee below does this n times, accumulating the kept elements into a list.

.. code-block:: scala

  def alternates[E](n: Int): IterV[E, List[E]] =
    drop1Keep1[E].
      replicate[List](n).
      foldRight(Done(List[Option[E]](),Empty))((x, y) => for {
        h <- x
        t <- y
      } yield h :: t).map(_.flatten)

Here’s an example run:

.. code-block:: scala

  scala> enumerate(List.range(1,15), alternates[Int](5)).run
  res85: List[Int] = List(2, 4, 6, 8, 10)

We can compose an iteratee with itself an infinite number of times, accumulating the results into a Stream until we hit the EOF:

.. code-block:: scala

  def repeat[E,A]: IterV[E,A] => IterV[E,Stream[A]] = i => {
    def step(s: Stream[A]): Input[E] => IterV[E, Stream[A]] = {
      case EOF => Done(s, EOF)
      case Empty => Cont(step(s))
      case El(e) => i match {
        case Done(a, f) => Done(s ++ Stream(a), El(e))
        case Cont(k) => for {
          h <- k(El(e))
          t <- repeat(i)
        } yield s ++ (h #:: t)
      }
    }
    Cont(step(Stream()))
  }

Here’s an example of that running, where we repeat the “alternates” iteratee from above:

.. code-block:: scala

  scala> enumerate(List.range(1,15), repeat(alternates[Int](5))).run.force
  res97: scala.collection.immutable.Stream[List[Int]] = Stream(List(2, 4, 6, 8, 10), List(12, 14))


File Input With Iteratees
-------------------------------------------------------------

Using the iteratees to read from file input turns out to be incredibly easy. The only difference is in how the data source is enumerated, and in order to remain lazy (and not prematurely perform any side-effects), we must return our iteratee in a monad:

.. code-block:: scala

  def enumReader[A](r: BufferedReader, it: IterV[String, A]): IO[IterV[String, A]] = {
    def loop: IterV[String, A] => IO[IterV[String, A]] = {
      case i@Done(_, _) => IO { i }
      case i@Cont(k) => for {
        s <- IO { r.readLine }
        a <- if (s == null) IO { i } else loop(k(El(s)))
      } yield a
    }
    loop(it)
  }


The monad being used here is an IO monad that I’ll explain in a second. The important thing to note is that the iteratee is completely oblivious to the fact that it’s being fed lines from a BufferedReader rather than a List.

Here is the IO monad I’m using. As you can see, it’s really just a lazy identity monad:

.. code-block:: scala

  object io {
    sealed trait IO[A] {
      def unsafePerformIO: A
    }

    object IO {
      def apply[A](a: => A): IO[A] = new IO[A] {
        def unsafePerformIO = a
      }
    }

    implicit val IOMonad = new Monad[IO] {
      def pure[A](a: => A): IO[A] = IO(a)
      def bind[A,B](a: IO[A], f: A => IO[B]): IO[B] = IO {
        implicitly[Monad[Function0]].bind(() => a.unsafePerformIO,
                                          (x:A) => () => f(x).unsafePerformIO)()
      }
    }
  }


To read lines from a file, we’ll do something like this:

.. code-block:: scala

  def bufferFile(f: File) = IO {
    new BufferedReader(new FileReader(f))
  }

  def closeReader(r: Reader) = IO {
    r.close
  }

  def bracket[A,B,C](init: IO[A], fin: A => IO[B], body: A => IO[C]): IO[C] =
  for { a <- init
        c <- body(a)
        _ <- fin(a) }
    yield c

  def enumFile[A](f: File, i: IterV[String, A]): IO[IterV[String, A]] =
    bracket(bufferFile(f),
            closeReader(_:BufferedReader),
            enumReader(_:BufferedReader, i))

The enumFile method uses bracketing to ensure that the file always gets closed. It’s completely lazy though, so nothing actually happens until you call unsafePerformIO on the resulting IO action:

.. code-block:: scala

  scala> enumFile(new File("/Users/runar/Documents/Iteratees.txt"), head) map (_.run)
  res2: io.IO[Option[String]] = io$IO@5f90b584

  scala> res2.unsafePerformIO
  res3: Option[String] = Some(Scalaz Tutorial: Enumeration-Based I/O With Iteratees)


That uses the “head” iteratee from above to get the first line of the file that I’m using to edit this blog post.

We can get the number of lines in two files combined, by composing two enumerations and using our “counter” iteratee from above:

.. code-block:: scala

  def lengthOfTwoFiles(f1: File, f2: File): IO[Int] = for {
    l1 <- enumFile(f1, counter)
    l2 <- enumFile(f2, l1)
  } yield l2.run


So what we have here is a uniform and compositional interface for enumerating both pure and effectful data sources. We can avoid holding on to the entire input in memory when we don’t want to, and we have complete control over when to stop iterating. The iteratee can decide whether to consume elements, leave them intact, or even truncate the input. The enumerator can decide whether to shut the iteratee down by sending it the EOF signal, or to leave it open for other enumerators.

There is even more to this approach, as we can use iteratees not just to read from data sources, but also to write to them. That will have to await another post.
