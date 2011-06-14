Type-Level Programming in Scala, Part 5b: Project Euler problem 4
------------------------------------------------------------------------

Posted by Mark on June 24, 2010

Let’s look at solving Project Euler problem #4 at the type level. Since we are working with binary numbers in Scala’s type system, we have to substantially reduce our expectations for what we can solve.

The first simplification is that we will look at palindromes in base 2 since our numbers are already in that representation. Also, we won’t be looking at large numbers. We’ll just find the largest binary palindrome that is the product of two binary numbers less than N. Even N = 15 takes a while to finish, so we’ll stick with 15.

I’ll show the approach by starting with normal Scala code that implements a brute force solution:

.. code-block:: scala

  object E4 {
    // class to hold the numbers i,j, prod = i*j
    case class Result(prod: Int, iMax: Int, jMax: Int)

    // compares Results based on the product
    implicit object ResultOrdering extends Ordering[Result] {
      def compare(a: Result, b: Result) = a.prod compare b.prod
    }

    // computes the largest binary palindrome made by multiplying
    //   two numbers <= start
    def apply(start: Int): Result =
      all(start).max

    // iterates over (i, j) pairs, calculating products, and keeping track of palindromes
    def all(start: Int) : Traversable[Result] =
      for(i <- (start to 1 by -1);
        j <- (i to 1 by -1);
        val prod = i*j
        if isPalindrome(prod) )
      yield
        Result(prod, i, j)

    // checks if the given integer is a binary palindrome
    def isPalindrome(i: Int) =
      i.toBinaryString == i.toBinaryString.reverse
  }

This is not easily translated directly to type-level programming, however. We need to convert it to explicitly recurse, explicitly track and update the maximum, and not use intermediate variables. We also drop the Result class in favor of a Tuple3.

.. code-block:: scala

  object E4_Explicit {

    type Result = (Int, Int, Int)
    def apply(start: Int): Result = apply( (1,1,1), start, start)

    def apply(max: Result, i: Int, j: Int): Result =
      apply1(max, (i*j, i, j))

    def apply1(max: Result, iteration: Result): Result =
      if(isPalindrome(iteration._1) && iteration._1 > max._1)
        apply2(iteration, iteration)
      else
        apply2(max, iteration)

    def apply2(max: Result, iteration: Result): Result =
      if(iteration._3 == 1)
      {
        if(iteration._2 == 1)
          max
        else
          apply(max, iteration._2 - 1, iteration._2 - 1)
      }
      else
        apply(max, iteration._2, iteration._3 - 1)

    def isPalindrome(i: Int) = i.toBinaryString == i.toBinaryString.reverse
  }

This still needs to be converted to better target type-level programming. Basically, ‘if’ statements and comparisons can be problematic. See the Expanded object in Euler4.scala for the normal Scala code closest to the type-level code that follows.

For the type-level code, let’s start with a test for binary palindromes:

.. code-block:: scala

  type isPalindrome[D <: Dense] = D#Reverse[DNil]#Compare[D]#eq

  println( toBoolean[ isPalindrome[_3] ] )
  // true
  println( toBoolean[ isPalindrome[_6] ] )
  // false
  println( toBoolean[ isPalindrome[_15] ] )
  // true

Then, let’s test if a number is a palindrome and greater than the current maximum palindrome:

.. code-block:: scala

  type isLargerPalindrome[D <: Dense, Max <: Dense] =
    isPalindrome[D] && D#Compare[Max]#gt

  println( toBoolean[ isLargerPalindrome[ _7, _3] ] )
  // true
  println( toBoolean[ isLargerPalindrome[ _3, _5] ] )
  // false
  println( toBoolean[ isLargerPalindrome[ _8, _5] ] )
  // false

What we want to do is then use Bool#If to process the result of isLargerPalindrome. Unfortunately, while everything worked ok up until this point, once we start building up the rest of the program, our type information gets lost. We have to do something less straightforward. We need to pass in what to return if the result is true or false. Roughly, we are inlining If, gt, and &&.

.. code-block:: scala

  // if D is a palindrome and D > Max, return T, otherwise F
  type p2[D <: Dense, Max <: Dense, T <: Up, F <: Up, Up] =
    p1[D, D#Compare[Max]#Match[F, F, T, Up], F, Up]

  // if D is a palindrome, return T, otherwise F
  type p1[D <: Dense, T <: Up, F <: Up, Up] =
    D#Reverse[DNil]#Compare[D]#Match[F, T, F, Up]

Other than this, we can do a more or less straightforward translation into type-level code. First, we have to make a parent trait that declares the signature of our recursing abstract type:

.. code-block:: scala

  trait Pali {
    // a type-level Tuple3
    type Result = Triple[Dense, Dense, Dense]

    type Apply[max <: Dense, iMax <: Dense, jMax <: Dense, i <: Dense, j <: Dense, P <: Pali] <: Result
  }

And then implement it.

.. code-block:: scala

  trait Pali0 extends Pali {
    // The entry point
    type App[start <: Dense, P <: Pali] = Apply[_1, _1, _1, start, start, P]

    // Recursion here
    type Apply[max <: Dense, iMax <: Dense, jMax <: Dense, i <: Dense, j <: Dense, P <: Pali] =
      Apply1[max, iMax, jMax, i, j, j#Mult[i], P ]

    type Apply1[max <: Dense, iMax <: Dense, jMax <: Dense, i <: Dense, j <: Dense, prod <: Dense, P <: Pali] =
      p2[prod, max,
        Apply2[prod, i, j, i, j#Dec, P],
        Apply2[max, iMax, jMax, i, j#Dec, P],
        Result ]

    type Apply2[max <: Dense, iMax <: Dense, jMax <: Dense, i <: Dense, j <: Dense, P <: Pali] =
      j#Match[
        Apply3[max, iMax, jMax, i, j, P],
        Apply3[max, iMax, jMax, i#Dec, i#Dec, P],
        Result]

    type Apply3[max <: Dense, iMax <: Dense, jMax <: Dense, i <: Dense, j <: Dense, P <: Pali] =
      i#Match[
        P#Apply[max, iMax, jMax, i, j, P],
        Triple.Apply[max, iMax, jMax, Dense],
        Result]

    // the palindrome tests from above
    type p1[D <: Dense, T <: Up, F <: Up, Up] =
      D#Reverse[DNil]#Compare[D]#Match[F, T, F, Up]
    type p2[D <: Dense, O <: Dense, T <: Up, F <: Up, Up] =
      p1[D, D#Compare[O]#Match[F, F, T, Up], F, Up]
  }

Running it gives:

.. code-block:: scala

  println( toTuple3[Pali0#App[_7, Pali0]] )
  // (21,7,3)

  // already pretty slow at _15
  println( toTuple3[Pali1.App[_15, Pali1.type]] )
  // (195,15,13)
