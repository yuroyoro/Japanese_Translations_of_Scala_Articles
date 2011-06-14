Part 2: implicitly と =:=
-------------------------------------------------------------

この記事は、型を比較するのにちょっとした便利なテクニックを紹介します(Jason Zauggがみせてくれました)。これらのテクニックは、のちほど型レベルの計算の結果を確認するために使用します。

``implicitly`` メソッドは、 ``scala.Predef`` に定義されています。

.. code-block:: scala

  def implicitly[T](implicit e: T): T = e

このメソッドは、スコープの中で型Tが持っている暗黙の値を計算するのに便利です。 この場合に我々が求めている暗黙の値は、AとBに対しての ``A =:= B`` という型です。 ``A =:= B`` はAとBが同じ型の時に見つかるでしょう。

例えば、これはコンパイルできます。

.. code-block:: scala

  scala> implicitly[Int =:= Int]
  res0: =:=[Int,Int] = <function1>


ですが、これはできません。

.. code-block:: scala

  scala> implicitly[Int =:= String]
  error: could not find implicit value for parameter e: =:=[Int,String]

同様に、 ``<:<`` と ``<%<`` もまた、それぞれ型が一致するか(conformance)、変換できるか(views)、を確認するために有効です。

.. note::

  conformance:        ``A <:< B`` は、AをBと見なせるか(AはBかそのサブタイプであるか)
  views(conversions): ``A <%< B`` は、(implicit conversionなどを用いて)AをBに変換できるか


conformance ( ``<:<`` )の例:

.. code-block:: scala

  scala> implicitly[Int =:= AnyVal]
  error: could not find implicit value for parameter e: =:=[Int,AnyVal]

  scala> implicitly[Int <:< AnyVal]
  res1: <:<[Int,AnyVal] = <function1>

conversion ( ``<%<`` )の例:

.. code-block:: scala

  scala> implicitly[Int <:< Long]
  error: could not find implicit value for parameter e: <:<[Int,Long]

  scala> implicitly[Int <%< Long]
  res1: <%<[Int,Long] = <function1>


コンパイラは ``<:<`` と ``<:<`` の型を表示するときに、中置記法を使いません。ちょっとしたパッチをコンパイラに当てることで中置記法にすることができますが、しかし


.. code-block:: scala

  scala> implicitly[Int <:< Long]
  error: could not find implicit value for parameter e: (Int <:< Long)

  scala> implicitly[Int <%< Long]
  res1: (Int <%< Long) = <function1>


これは単純名が曖昧な場合に完全名を出すようにするちょっとしたハックです。2.8.0.finalがリリースされたら、コンパイラにこの変更が取り込まれるように改善しようと思っています。

.. note::

  訳注) 2.9.0.1の段階でもまだこの変更は入っていない...。


.. TODO:: バッチの場所さがす


