Part 4d: ペアノ数論(Peano arithmetic)
-------------------------------------------------------------

今、 ``Nat#FoldR`` を型レベル自然数における計算を定義するために利用しようとしています。

型レベル関数とfoldは、値レベルの関数の形式に従います(多くの場合で、単にn回関数を適用するだけで現在の値に対してはなにも行いません)。


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

参考までに、 ``Fold`` 型はこのように定義しました。

.. code-block:: scala

  trait Fold[-Elem, Value] {
     type Apply[N <: Elem, Acc <: Value] <: Value
  }


加算はこうなります。

.. code-block:: scala

   type Add[A <: Nat, B <: Nat] = A#FoldR[B, Nat, Inc]
   type Inc = Fold[Nat, Nat] {
      type Apply[N <: Nat, Acc <: Nat] = Succ[Acc]
   }


乗算は、加算を応用して定義できます。

.. code-block:: scala

   type Mult[A <: Nat, B <: Nat] = A#FoldR[_0, Nat, Sum[B]]
   type Sum[By <: Nat] = Fold[Nat, Nat] {
      type Apply[N <: Nat, Acc <: Nat] = Add[By, Acc]
   }


階乗は、乗算を用いて定義されます。これは、いかなる理由であれ、fold-leftでしか動作しません。

.. code-block:: scala

   type Fact[A <: Nat] = A#FoldL[_1, Nat, Prod]
   type Prod = Fold[Nat, Nat] {
      type Apply[N <: Nat, Acc <: Nat] = Mult[N, Acc]
   }


冪乗もまた、乗算を用います。

.. code-block:: scala

   type Exp[A <: Nat, B <: Nat] = B#FoldR[_1, Nat, ExpFold[A]]
   type ExpFold[By <: Nat] = Fold[Nat, Nat] {
      type Apply[N <: Nat, Acc <: Nat] = Mult[By, Acc]
   }


剰余は、 ``Compare#eq`` を使います。


.. code-block:: scala

   type Mod[A <: Nat, B <: Nat] = A#FoldR[_0, Nat, ModFold[B]]
   type ModFold[By <: Nat] = Fold[Nat, Nat] {
      type Wrap[Acc <: Nat] = By#Compare[Acc]#eq
      type Apply[N <: Nat, Acc <: Nat] = Wrap[Succ[Acc]]#If[_0, Succ[Acc], Nat]
   }


ここで、

.. code-block:: scala

   def toInt[N <: Nat] : Int


``toBoolean`` のように、 ``toInt`` を定義します。しかしながら、大きい数値に対しては不便ですので、主にこちらを用います。

.. code-block:: scala

  type Eq[A <: Nat, B <: Nat] = A#Compare[B]#eq
  toBoolean[ Eq[ A, B ] ]


いくつかの例です。

.. code-block:: scala

   type Sq[N <: Nat] = Exp[N, _2]

   val true = toBoolean[ Eq[ Sq[_9], Add[_1,Mult[_8,_10]] ] ]

   val true = toBoolean[ Eq[ Sq[Sq[_9]], Sq[Add[_1,Mult[_8,_10]]] ] ]

   val true = toBoolean[ Eq[ Mod[ Exp[_9,_4], _6], _3] ]

   val true = toInt[ Mod[ Sq[_9], _6] ] == 81 % 6

Operations on Nat are simple to implement using our folds, but don’t work well for reasonably large numbers or more complicated expressions. Around 10,000, the compilation time increases a lot or the stack overflows. Next, we’ll look at representing unsigned integers in binary and (sort of) solve Euler problem #4.


``Nat`` における演算はシンプルにfoldを用いた実装ですが、いくつかの理由により大きい数値や複雑な式では上手くどうさしません。大体10,000くらいで、コンパイル時間が増加するかスタックオーバーフローが起こります。

次では、符号無し整数の二進表現を提示し、Project-Eulerの#4の問題を解いてみます。
