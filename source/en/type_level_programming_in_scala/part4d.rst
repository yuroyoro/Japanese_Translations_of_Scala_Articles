Type-Level Programming in Scala, Part 4d: Peano arithmetic
-------------------------------------------------------------

Posted by Mark on June 21, 2010

Now we will use Nat#FoldR to define arithmetic on type-level natural numbers.

Our type level functions and folds will follow the form of these value level functions. (In most cases, we are really just applying a function n times and not doing anything with the current iteration value.)

.. code-block:: scala

  val list(n: Int) = (n to 1 by -1).toList

  // like Succ[N]
  def succ(a: Int) = a+1

  // like Add[A <: Nat, B <: Nat]
  def add(a: Int, b: Int) =
     ( list(a) :\ b) { (n, acc) => succ(acc) }

  def mult(a: Int, b: Int) =
     ( list(a) :\ 0) { (n, acc) => add(acc, b) }

  def exp(a: Int, b: Int) =
     (list(b) :\ 1 ) { (n, acc) => mult(acc, a) }

  def fact(a: Int) =
     (list(a) :\ 1) { (n, acc) => mult(n, acc) }

  def mod(a: Int, b: Int) =
     (list(a) :\ 0 ) { (n, acc) => if(acc+1 == b) 0 else acc+1 }

For reference, the Fold type was defined as:

.. code-block:: scala

  trait Fold[-Elem, Value] {
     type Apply[N <: Elem, Acc <: Value] <: Value
  }

Addition looks like:

.. code-block:: scala

   type Add[A <: Nat, B <: Nat] = A#FoldR[B, Nat, Inc]
   type Inc = Fold[Nat, Nat] {
      type Apply[N <: Nat, Acc <: Nat] = Succ[Acc]
   }

Multiplication is defined in terms of addition:

.. code-block:: scala

   type Mult[A <: Nat, B <: Nat] = A#FoldR[_0, Nat, Sum[B]]
   type Sum[By <: Nat] = Fold[Nat, Nat] {
      type Apply[N <: Nat, Acc <: Nat] = Add[By, Acc]
   }

Factorial is defined in terms of multiplication. For whatever reason, it only worked with a fold left:

.. code-block:: scala

   type Fact[A <: Nat] = A#FoldL[_1, Nat, Prod]
   type Prod = Fold[Nat, Nat] {
      type Apply[N <: Nat, Acc <: Nat] = Mult[N, Acc]
   }

Exponentiation is also in terms of multiplication:

.. code-block:: scala

   type Exp[A <: Nat, B <: Nat] = B#FoldR[_1, Nat, ExpFold[A]]
   type ExpFold[By <: Nat] = Fold[Nat, Nat] {
      type Apply[N <: Nat, Acc <: Nat] = Mult[By, Acc]
   }

Mod uses Compare#eq:

.. code-block:: scala

   type Mod[A <: Nat, B <: Nat] = A#FoldR[_0, Nat, ModFold[B]]
   type ModFold[By <: Nat] = Fold[Nat, Nat] {
      type Wrap[Acc <: Nat] = By#Compare[Acc]#eq
      type Apply[N <: Nat, Acc <: Nat] = Wrap[Succ[Acc]]#If[_0, Succ[Acc], Nat]
   }

We can define:

.. code-block:: scala

   def toInt[N <: Nat] : Int

similar to toBoolean. However, it is not very usable for large integers, so we will mainly use:

.. code-block:: scala

  type Eq[A <: Nat, B <: Nat] = A#Compare[B]#eq
  toBoolean[ Eq[ A, B ] ]

Some examples:

.. code-block:: scala

   type Sq[N <: Nat] = Exp[N, _2]

   val true = toBoolean[ Eq[ Sq[_9], Add[_1,Mult[_8,_10]] ] ]

   val true = toBoolean[ Eq[ Sq[Sq[_9]], Sq[Add[_1,Mult[_8,_10]]] ] ]

   val true = toBoolean[ Eq[ Mod[ Exp[_9,_4], _6], _3] ]

   val true = toInt[ Mod[ Sq[_9], _6] ] == 81 % 6

Operations on Nat are simple to implement using our folds, but don’t work well for reasonably large numbers or more complicated expressions. Around 10,000, the compilation time increases a lot or the stack overflows. Next, we’ll look at representing unsigned integers in binary and (sort of) solve Euler problem #4.
