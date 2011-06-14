Part 1: 再帰型(Type Recursion)
-------------------------------------------------------------

Scala2.8の型システムは再帰を許可し、[1, 2]で示すような無限再帰が可能です。任意の再帰は許可されておらず、不正な再帰はエラーとなります。再帰が許可されていることは、実際にはバグなのでしょうが、だからこそいくつかの興味深い挑戦が可能です。

再帰を使うには、type宣言を持ったトレイトを定義します。type宣言は型パラメータと境界を持った抽象型として宣言します。そして、そのトレイトを継承したサブトレイトで、抽象型を実装します。この実装では再帰は許可されました。その際の制約については後ほど議論します。

無限ループとなる例:

.. code-block:: scala

  // 抽象型を持つトレイト
  trait Recurse {
    type Next <: Recurse
    // このtype宣言が再帰関数にあたる
    type X[R <: Recurse] <: Int
  }
  // implementation
  trait RecurseA extends Recurse {
    type Next = RecurseA
    // 再帰関数の"実装"
    type X[R <: Recurse] = R#X[R#Next]
  }
  object Recurse {
    // 無限ループ...
    type C = RecurseA#X[RecurseA]
  }

.. note::

  明示的にInt型に指定した型は、任意の型に置き換えることが可能です。一般的に、この境界は何らかの意味を持っています。実際の型パラメータに何を与えられたか知ることなく、結果がどのようなものかある程度知ることができます。

再帰を可能とするためには、いくつかのルールがあります。上限境界を関数の型パラメータに指定できますが、再帰の最中に変更する事はできません。

注目してください。

.. code-block:: scala

  trait Recurse {
    type X[A, B] <: A
  }

型パラメータ ``A`` と ``B`` について。 ``Recurse`` のサブトレイトにおける ``X`` の実装で、 ``A`` は再帰している間は変更できませんが、 ``B`` は変更可能です。具体的な例はこのあとの型レベルの数値、HList、二分探索木を扱う記事でお見せします。


.. TODO:: type-projectionどう訳す?

もうひとつの制約は、あらゆる再帰はtype-projectionのようなプレフィックスを持たなければならない、ということです。

.. code-block:: scala

  type X[R <: Recurse] = R#X[R#Next]

このように、直接的に今扱っている型の型メンバーへ再帰させることはできないのです。

.. code-block:: scala

  type X[R <: Recurse] = X[R#Next]


このような妥当な制約のもとで、型システム上ので再帰およびcorecursionを実現できるので、 完全な代数的セマンティックスを表現できます。Scalaの型システムはチューリング完全なプログラミング言語と言えます。


- [1] http://apocalisp.wordpress.com/2009/09/02/more-scala-typehackery/
- [2] http://michid.wordpress.com/2010/01/29/scala-type-level-encoding-of-the-ski-calculus/

