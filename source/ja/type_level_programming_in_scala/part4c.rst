Part 4c: ペアノ数における一般的な再帰
--------------------------------------------------------------------------------

算術演算を定義する際に核となる演算は、foldにより数値のlistをNから _1 まで下っていくことです( _0 は空リストのように扱われます)。fold leftとfold rightを用いることで、このように加算と乗算の演算を定義できます。

``FoldR`` 型を ``Nat`` に追加します。


.. code-block:: scala

  sealed trait Nat {
    type FoldR[Init <: Type, Type, F <: Fold[Nat, Type]] <: Type
  }

ここで

.. code-block:: scala

  trait Fold[-Elem, Value] {
    type Apply[E <: Elem, V <: Value] <: Value
  }


``Fold`` は ``Elem`` と ``Value`` に対して定義されており、サブタイプで ``Elem`` と ``Value`` を引数にとって ``Value`` を返す関数型となります。この ``Apply`` 関数の型を、(以下のようなトレイトで表すことのできる) ``List.foldRight`` と比較してみてください。

.. code-block:: scala

  trait Fold[-Elem, Value] {
    def apply(e: Elem, v: Value): Value
  }


データ構造から、 ``Elem`` 型の値と ``Value`` 型に集積された値から、新しい ``value`` 型の値に変換します。 型レベルFoldにおいて、 ``E`` と ``V`` は引数eとvの役割を果たします。

もういちど、FoldRの定義を示します。

.. code-block:: scala

  type FoldR[Init <: Type, Type, F <: Fold[Nat, Type]] <: Type


最初の型引数の ``Init`` 型とその後計算される結果がすべて ``Type`` のサブタイプになることがわかります。これがわかりにくいのなら、先ほどのList上のfoldと見比べてみるとよいでしょう。 ``F`` は ``Fold`` のサブタイプで、このfold-rightで使用するものです。これはFoldRに与えられる型レベルの関数です。またも、List.foldRightで与えられた関数と相似しています。最後に、結果の型はTypeのサブタイプとなります。

あらかじめ触れておいたように、型パラメータとして宣言した型の制約については、再帰の最中に変更する事はできません。そのため ``Type`` は再帰を通じて同一である必要があります。次から、FoldRの実装を見ていきましょう。

``_0`` の実装は簡単です。 ``Nil`` での foldR と同様、単に初期値として与えられた型を返すだけです。


.. code-block:: scala

  sealed trait _0 extends Nat {
    type FoldR[Init <: Type, Type, F <: Fold[Nat, Type]] =
      Init
  }


このように、動作することが確認できます


.. code-block:: scala

  type C = _0#FoldR[ Int, AnyVal, Fold[Nat, AnyVal]]

  // C はIntなのでコンパイルできる
  implicitly[ C =:= Int ]

  // Cは AnyVal ではないのでコンパイれない
  //implicitly[ C =:= AnyVal ]


もうしこし興味深いケースとして、Succにおける実装を示します

.. code-block:: scala

  sealed trait Succ[N <: Nat] extends Nat {
    type FoldR[Init <: Type, Type, F <: Fold[Nat, Type]] =
      F#Apply[Succ[N], N#FoldR[Init, Type, F]]
  }


この部分は、最初にひとつ前の数で再帰を行います。

.. code-block:: scala

  N#FoldR[Init, Type, F]


型レベル再帰において、 ``Type`` は変更せずに渡されることに注意してください。初期値の ``Init`` と ``Fold`` も同様です。 すでに、 ``N`` が ``_0`` であるときには ``Init`` が返されるのを見ました。 ので、 ``Succ[_0]`` では ``Fold`` 型の ``F`` に自身( ``Succ[_0]`` )と初期値 ``Init`` を適用することになります。 ``Succ[N]`` は、あたかもfold-rightのように、次々と ``F#Apply`` を適用しつづけるでしょう。

面白いのは、型安全なままで何かを引き出すことができたということです。次の記事では、 ``FoldR`` におけるAdd、Mult、Exp、およびModを定義するつもりです。

