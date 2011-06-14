Simple SKI Combinator Calculus in Scala’s Type System
-------------------------------------------------------------

Posted by Rúnar on January 13, 2011

I would like to make two claims:

The Scala type system is turing complete.
This is not a bug.
The SKI combinator calculus is known to be turing equivalent:

.. code-block:: scala

  trait λ { type ap[_<:λ]<:λ }
  type I = λ{type ap[X<:λ] = X }
  type K = λ{type ap[X<:λ] = λ{type ap[Y<:λ] = X }}
  type S = λ{type ap[X<:λ] = λ{type ap[Y<:λ] = λ{type ap[Z<:λ] = X#ap[Z]#ap[Y#ap[Z]] }}}

And we can blow up the compiler with unbounded recursion, without resorting to circular definitions:

.. code-block:: scala

  type Y = S#ap[I]#ap[I]#ap[S#ap[I]#ap[I]]

The latter should crash the compiler with a stack overflow error.
