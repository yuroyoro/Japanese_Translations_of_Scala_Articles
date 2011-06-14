Part 4a: ペアノ数(Peano number)の基礎
--------------------------------------------------------------------------

最初の記事で触れたように、Scalaの型レベルでの再帰が可能です。この最初のアプリケーションは、型システム上の数値表現(ペアノ数)です。例えば、この後でやる ``HList`` でのインデックス指定で使われます。もちろん基本的な考え方は新しいものではなく、この記事のために必要な道具はすでに構築済みです。

See also:

- http://www.haskell.org/haskellwiki/Peano_numbers
- http://trac.assembla.com/metascala/browser/src/metascala/Nats.scala

基本的な型レベルの非負の整数表現は、こうです。

.. code-block:: scala

  sealed trait Nat
  sealed trait _0 extends Nat
  sealed trait Succ[N <: Nat] extends Nat

``Succ`` を用いてより大きな整数を構成できます。

.. code-block:: scala

  type _1 = Succ[_0]
  type _2 = Succ[_1]
  ...

``Bool`` のように、 ``Nat`` というなんらかの構造を定義しました。ここでは、全ての型レベルでコアとなる3つの演算を定義します。

この演算は、 ``Bool`` に ``If`` を定義したように、 ``HList`` のインデックス指定に必要です。これを ``Match`` と呼びます。その定義は、

.. code-block:: scala

  sealed trait Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] <: Up
  }
  sealed trait _0 extends Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] = IfZero
  }
  sealed trait Succ[N <: Nat] extends Nat {
     type Match[NonZero[N <: Nat] <: Up, IfZero <: Up, Up] = NonZero[N]
  }

これらは、 ``Nat`` が ``Succ`` の時に型コンストラクタをとることを除いて、 ``Bool#If`` のように見えます。 ``Succ[N}`` に対して ``Match`` が呼び出されると、結果は ``NonZero[N]`` となります。もし実際の ``Nat`` が ``_0`` ならば、 ``IfZero`` が返されます。 ``Up`` パラメータは、 ``Bool#If`` のように結果に対する上限境界です。

これは、 ``Nat`` を解体しているかのように見えます。例えば、パラメータが ``_0`` であるか決定する「型」を定義することがでます。

.. code-block:: scala

  type Is0[A <: Nat] = A#Match[ConstFalse, True, Bool]
  type ConstFalse[A] = False


もし ``A`` が ``_0`` ならば、 ``A#Match`` の結果として ``True`` が返ります。 ``A`` が ``Succ[N]`` であれば、 ``ConstFalse[N]`` が結果です。 ``ConstFalse`` は常に ``False`` となるので、 ``Succ`` に対する結果は ``False`` となります。


``Match`` は ``HList`` のために必要な演算です。ともあれ、続けて比較・加算・乗法・剰余・冪法を楽しみながら定義してみましょう。
