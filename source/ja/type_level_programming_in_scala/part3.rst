Part 3: 真偽値(Boolean)
-------------------------------------------------------------


.. TODO:: Church encoding ?

型レベルプログラミングを紹介するときのよい導入は、真偽値(boolean)をチャーチ記号化することです。これはいかなる過剰な空想を求めないですし、そしてbooleanは数値を比較する際に必要です。
もしあなたがすでに、真偽値のチャーチ化に慣れ親しんでいるなら、特にすでにScalaで見たことがあるのなら、この記事は特筆に値することはありません。この記事と次の型レベルのペアノ数の記事は、型レベルプログラミングに慣れていない人のためのものです。

See also:

- http://svn.assembla.com/svn/metascala/src/metascala/Booleans.scala.

.. code-block:: scala

   sealed trait Bool
   sealed trait True extends Bool
   sealed trait False extends Bool

条件判断を表す式として、 ``Bool`` 型に抽象型メンバー ``If`` を追加することができます。
``If`` は3つの型パラメータを取ります。  ``Bool`` が ``True`` だった時の型、 ``False`` だったときの型、そしてその2つの上限境界となる型です。上限境界は、 ``If`` の結果を使うときにしばしば重要となります。

拡張された ``Bool`` 型は、このようになります。

.. code-block:: scala

  sealed trait Bool {
   type If[T <: Up, F <: Up, Up] <: Up
  }

``True`` と ``False`` はそれぞれ適した型を単に返します。

.. code-block:: scala

  sealed trait True extends Bool {
   type If[T <: Up, F <: Up, Up] = T
  }

  sealed trait False extends Bool {
   type If[T <: Up, F <: Up, Up] = F
  }

このように使います。

.. code-block:: scala

  scala> type Rep[A <: Bool] = A#If[ Int, Long, AnyVal ]
  defined type alias Rep

  scala> implicitly[ Rep[True] =:= Int ]
  res1: =:=[Rep[Booleans.True],Int] = <function1>

  scala> implicitly[ Rep[False] =:= Int ]
  error: could not find implicit value for parameter e: =:=[Rep[Booleans.False],Int]

  scala> implicitly[ Rep[False] =:= Long ]
  res3: =:=[Rep[Booleans.False],Long] = <function1>

いくつかの追加の抽象型を ``Bool`` に追加しました。

.. code-block:: scala

  object Bool {
   type &&[A <: Bool, B <: Bool] = A#If[B, False, Bool]
   type || [A <: Bool, B <: Bool] = A#If[True, B, Bool]
   type Not[A <: Bool] = A#If[False, True, Bool]
  }

使用例です。

.. code-block:: scala

  scala> import Bool._
  import Bool._

  scala> implicitly[ True && False || Not[False] =:= True ]
  res1: =:=[Bool.||[Bool.&&[True,False],Bool.Not[False]],True] = <function1>


直接表示するために、 ``Bool型`` を ``Bool値`` に変換するメソッドを ``Bool型`` に追加することもできます。


.. code-block:: scala

 def toBoolean[B <: Bool](implicit b: BoolRep[B]): Boolean = b.value

 class BoolRep[B <: Bool](val value: Boolean)
 implicit val falseRep: BoolRep[False] = new BoolRep(false)
 implicit val trueRep: BoolRep[True] = new BoolRep(true)

例えば、

.. code-block:: scala

  scala> toBoolean[ True && False || Not[False] ]
  res0: Boolean = true

これは、型レベルの計算結果をチェックするもう一つのメソッドです。

次は、型レベルのペアノ数です。

