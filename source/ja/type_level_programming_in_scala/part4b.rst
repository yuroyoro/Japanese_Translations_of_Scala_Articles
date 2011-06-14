Part 4b: ペアノ数の比較
-----------------------------------------------------------------------------

次に、自然数の比較を見てみましょう。 ``Comparison型`` をこのように定義します。


.. code-block:: scala

  sealed trait Comparison {
    type Match[IfLT <: Up, IfEQ <: Up, IfGT <: Up, Up] <: Up

    type gt = Match[False, False, True, Bool]
    type ge = Match[False, True, True, Bool]
    type eq = Match[False, True, False, Bool]
    type le = Match[True, True, False, Bool]
    type lt = Match[True, False, False, Bool]
  }

実装はこうです。

.. code-block:: scala

  sealed trait GT extends Comparison {
    type Match[IfLT <: Up, IfEQ <: Up, IfGT <: Up, Up] = IfGT
  }
  sealed trait LT extends Comparison {
    type Match[IfLT <: Up, IfEQ <: Up, IfGT <: Up, Up] = IfLT
  }
  sealed trait EQ extends Comparison {
    type Match[IfLT <: Up, IfEQ <: Up, IfGT <: Up, Up] = IfEQ
  }


``Nat型`` の抽象型として ``Compare型`` を定義します。前回定義した ``Match`` とともに以下のようになります。

.. code-block:: scala

  sealed trait Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] <: Up

     type Compare[N <: Nat] <: Comparison
  }


``_0#Compare`` では、 結果が ``EQ`` となるのは比較先が ``_0`` であるときで、そうでなければ ``LT`` です。

.. code-block:: scala

  sealed trait _0 extends Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] = IfZero

     type Compare[N <: Nat] =
        N#Match[ConstLT, EQ, Comparison]

     type ConstLT[A] = LT
  }


``Succ[N]#Compare`` では、 もしもう一方の ``Nat`` が ``_0`` ならば 結果は ``GT`` となり、そうで無ければ ``O-1`` から ``N`` までの再帰的な比較を行います( ``N`` は ``Succ[N]`` - 1 であることを思い出してください)。

.. code-block:: scala

  sealed trait Succ[N <: Nat] extends Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] = NonZero[N]

     type Compare[O <: Nat] =
        O#Match[N#Compare, GT, Comparison]
  }

ここで再帰が可能なのは、 ``Compare`` が親トレイトの抽象型の実装であり、上限境界(ここでは ``Comparison`` )が再帰の間に変化しないからです。 ``N#Compare`` のような呼び出しを行っているのは、直接的な再帰ができないからです。

例です:

.. code-block:: scala

  scala> toBoolean[ _0#Compare[_0]#eq ]
  res0: true

  scala> toBoolean[ _0#Compare[_0]#lt ]
  res1: false

  scala> toBoolean[ _3#Compare[_4]#le ]
  res2: true


二分探索木において、 ``Compare`` により自然数( ``Nat`` )をツリーのkeyとして用いることができるようになります。次は、 ``Nat`` で一般的な再帰をfoldを用いて定義してみます。

